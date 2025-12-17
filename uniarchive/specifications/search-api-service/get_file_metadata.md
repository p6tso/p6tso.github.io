# get_file_metadata

## 1. Валидация входных параметров

| Параметр | Источник | Тип данных | Обязательность | Проверка |
|----------|----------|------------|----------------|----------|
| `file_id` | Path parameter | String | Обязательно | Формат UUID v4 |
| `include_technical` | Query string | Boolean | Опционально | true/false, по умолчанию false |
| `include_storage_info` | Query string | Boolean | Опционально | true/false, по умолчанию false (только для админов) |

## 2. Запрос метаданных из БД

**Основной SQL запрос:**
```sql
SELECT 
    -- Основные метаданные
    f.id, f.original_name, f.subject, f.uploaded_at,
    f.file_size, f.mime_type, f.uploaded_by,
    
    -- Технические метаданные (только при include_technical=true)
    CASE WHEN :include_technical THEN f.storage_path ELSE NULL END as storage_path,
    CASE WHEN :include_technical THEN f.checksum ELSE NULL END as checksum,
    
    -- Статус обработки
    ft.processing_status, ft.processed_at, ft.error_message,
    ft.text_length,
    
    -- Аудит (только при include_technical=true)
    CASE WHEN :include_technical THEN 
        (SELECT COUNT(*) FROM files_audit fa WHERE fa.file_id = f.id)
    ELSE NULL END as audit_count,
    
    -- Статистика доступа
    f.download_count, f.last_downloaded_at
    
FROM files f
LEFT JOIN file_texts ft ON f.id = ft.file_id
WHERE f.id = :file_id
```

**Примечание:** Предполагается, что в таблицу `files` добавлены поля:
- `checksum VARCHAR(64)` - контрольная сумма файла
- `download_count INTEGER DEFAULT 0` - количество скачиваний
- `last_downloaded_at TIMESTAMP` - время последнего скачивания

## 3. Запрос дополнительной информации из MinIO

**Если `include_storage_info=true` и пользователь - администратор:**
1. Получение информации об объекте из MinIO:
    - `last_modified`
    - `etag` (MD5 хеш)
    - `version_id` (если включено versioning)
    - Размер в хранилище
    - MIME-тип из метаданных объекта
2. Проверка целостности:
    - Сравнение `etag` с `checksum` из БД
    - Сравнение размера в MinIO с `file_size` из БД

## 4. Сбор информации из аудита

**Если `include_technical=true`:**
```sql
SELECT 
    changed_at, changed_by, operation,
    jsonb_pretty(old_data) as old_data_preview,
    jsonb_pretty(new_data) as new_data_preview
FROM files_audit 
WHERE file_id = :file_id
ORDER BY changed_at DESC
LIMIT 10
```

## 5. Форматирование ответа

### Базовый ответ (для обычных пользователей):
```json
{
  "metadata": {
    "id": "uuid",
    "original_name": "string",
    "subject": "string",
    "uploaded_at": "ISO 8601",
    "file_size": 1234567,
    "file_size_human": "1.23 MB",
    "mime_type": "application/pdf",
    "uploaded_by": "user123 (анонимизировано)",
    "checksum": null,  // скрыто
    "storage_path": null  // скрыто
  },
  "processing": {
    "status": "completed",
    "processed_at": "ISO 8601",
    "error_message": null,
    "text_length": 3421,
    "text_length_human": "3.4 KB"
  },
  "access": {
    "download_count": 15,
    "last_downloaded_at": "ISO 8601",
    "days_since_upload": 7
  },
  "links": {
    "self": "/api/v1/files/{file_id}/metadata",
    "file": "/api/v1/files/{file_id}",
    "download": "/api/v1/files/{file_id}/download",
    "status": "/api/v1/upload/{file_id}/status"
  }
}
```

### Расширенный ответ (с `include_technical=true`):
```json
{
  ...базовые поля...,
  "technical": {
    "storage": {
      "path": "uniarchive-files/2024/01/uuid.pdf",
      "bucket": "uniarchive-files",
      "checksum": "sha256:abc123...",
      "checksum_verified": true,
      "storage_size": 1234567,
      "size_match": true
    },
    "database": {
      "record_created": "ISO 8601",
      "record_updated": "ISO 8601",
      "row_size_estimate": "15 KB"
    },
    "audit": {
      "total_events": 3,
      "recent_events": [
        {
          "timestamp": "ISO 8601",
          "user": "admin",
          "operation": "UPDATE",
          "field_changed": "processing_status"
        }
      ]
    }
  }
}
```

### Административный ответ (с `include_storage_info=true`):
```json
{
  ...расширенные поля...,
  "storage_detailed": {
    "minio_info": {
      "object_name": "2024/01/uuid.pdf",
      "last_modified": "ISO 8601",
      "etag": "\"abc123def456\"",
      "version_id": "null или uuid",
      "content_type": "application/pdf",
      "metadata": {
        "original-filename": "Лекция 1.pdf",
        "x-amz-meta-subject": "Математика"
      }
    },
    "verification": {
      "checksum_match": true,
      "size_match": true,
      "metadata_match": true,
      "last_verified": "ISO 8601"
    },
    "urls": {
      "presigned_download": "https://minio/... (TTL 5 min)",
      "presigned_metadata": "https://minio/... (TTL 1 min)"
    }
  }
}
```

## Ошибки

| Код ошибки | Условие | HTTP статус | Сообщение |
|------------|---------|-------------|-----------|
| `INVALID_UUID` | Некорректный file_id | 400 | "Invalid file identifier format" |
| `FILE_NOT_FOUND` | Файл не существует | 404 | "File metadata not found" |
| `PERMISSION_DENIED` | Недостаточно прав для technical/storage info | 403 | "Insufficient permissions for requested metadata level" |
| `DATABASE_ERROR` | Ошибка запроса к БД | 503 | "Metadata service unavailable" |
| `STORAGE_ERROR` | Ошибка запроса к MinIO | 503 | "Storage metadata unavailable" |
| `INCONSISTENT_DATA` | Несоответствие данных БД и MinIO | 500 | "Data inconsistency detected" |


