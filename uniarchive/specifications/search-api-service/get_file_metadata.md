# get_file_metadata

## 1. Валидация входных параметров

| Параметр | Источник | Тип данных | Обязательность | Проверка |
|----------|----------|------------|----------------|----------|
| `file_id` | Path parameter | String | Обязательно | Формат UUID v4 |

## 2. Запрос метаданных из БД

**Основной SQL запрос:**
```sql
SELECT 
    -- Основные метаданные
    f.id, f.original_name, f.subject, f.uploaded_at,
    f.file_size, f.mime_type, 
    
    -- Технические метаданные (только при include_technical=true)
    CASE WHEN :include_technical THEN f.storage_path ELSE NULL END as storage_path,
    CASE WHEN :include_technical THEN f.checksum ELSE NULL END as checksum,
    
    -- Статус обработки
    ft.processing_status, ft.processed_at, ft.error_message,
    ft.text_length,

    
FROM files f
LEFT JOIN file_texts ft ON f.id = ft.file_id
WHERE f.id = :file_id
```


## 3. Форматирование ответа

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

## Ошибки

| Код ошибки | Условие | HTTP статус | Сообщение |
|------------|---------|-------------|-----------|
| `INVALID_UUID` | Некорректный file_id | 400 | "Invalid file identifier format" |
| `FILE_NOT_FOUND` | Файл не существует | 404 | "File metadata not found" |
| `PERMISSION_DENIED` | Недостаточно прав для technical/storage info | 403 | "Insufficient permissions for requested metadata level" |
| `DATABASE_ERROR` | Ошибка запроса к БД | 503 | "Metadata service unavailable" |
| `STORAGE_ERROR` | Ошибка запроса к MinIO | 503 | "Storage metadata unavailable" |
| `INCONSISTENT_DATA` | Несоответствие данных БД и MinIO | 500 | "Data inconsistency detected" |


