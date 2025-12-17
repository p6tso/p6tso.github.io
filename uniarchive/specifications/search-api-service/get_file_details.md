# get_file_details

## 1. Валидация входных параметров

| Параметр | Источник | Тип данных | Обязательность | Проверка |
|----------|----------|------------|----------------|----------|
| `file_id` | Path parameter | String | Обязательно | Формат UUID v4 |
| `include_text` | Query string | Boolean | Опционально | true/false, по умолчанию false |

## 2. Построение SQL запроса

**Базовый SQL запрос (без текста):**
```sql
SELECT 
    f.id, f.original_name, f.storage_path, f.subject, 
    f.uploaded_at, f.file_size, f.mime_type, f.uploaded_by,
    ft.processing_status, ft.processed_at, ft.error_message,
    CASE 
        WHEN ft.extracted_text IS NOT NULL THEN LENGTH(ft.extracted_text)
        ELSE 0 
    END as text_length
FROM files f
LEFT JOIN file_texts ft ON f.id = ft.file_id
WHERE f.id = :file_id
```

**Расширенный запрос (с текстом):**
```sql
-- Добавляется при include_text=true
SELECT ft.extracted_text
FROM file_texts ft 
WHERE ft.file_id = :file_id AND ft.processing_status = 'completed'
```

## 3. Обработка результата запроса

**Последовательные операции:**
1. Выполнение основного запроса
2. Если результат пустой → файл не найден
3. Если `include_text=true` и `processing_status='completed'`:
    - Выполнение второго запроса для получения текста
    - Ограничение длины текста в ответе (первые 5000 символов)
4. Форматирование дат в ISO 8601
5. Расчет производных полей:
    - `days_since_upload = CURRENT_DATE - uploaded_at`
    - `is_processed = processing_status = 'completed'`
    - `has_error = processing_status = 'failed'`


## 5. Формирование ответа

### Основная структура ответа:
```json
{
  "file": {
    "id": "uuid",
    "original_name": "string",
    "subject": "string",
    "uploaded_at": "ISO 8601",
    "file_size": 123456,
    "mime_type": "application/pdf",
    "uploaded_by": "string или null",
    "storage_info": {
      "path": "string (только для админов)",
      "bucket": "uniarchive-files",
      "verified": true/false
    }
  },
  "processing": {
    "status": "pending|processing|completed|failed",
    "processed_at": "ISO 8601 или null",
    "error_message": "string или null",
    "text_length": 1234,
    "text_preview": "первые 500 символов или null"
  },
  "calculated": {
    "days_since_upload": 5,
    "is_processed": true/false,
    "has_error": true/false,
    "download_size_mb": 1.23
  },
  "links": {
    "self": "/api/v1/files/{file_id}",
    "download": "/api/v1/files/{file_id}/download",
    "metadata": "/api/v1/files/{file_id}/metadata",
    "status": "/api/v1/upload/{file_id}/status"
  },
  "metadata": {
    "query_time_ms": 45,
    "text_included": true/false,
    "timestamp": "ISO 8601"
  }
}
```

### С текстом (include_text=true):
```json
{
  ...основные поля...,
  "processing": {
    ...,
    "extracted_text": "полный текст или null",
    "text_truncated": true/false (если обрезано до 5000 символов)
  }
}
```

## 6. Проверка прав доступа

**Уровни доступа:**
1. **Публичный доступ:** Все поля кроме `storage_path` и `uploaded_by`
2. **Владелец файла:** Все поля, включая `uploaded_by`
3. **Администратор:** Все поля, включая `storage_path`

**Логика проверки:**
```python
if current_user.is_admin:
    # Показать все
elif current_user.id == file.uploaded_by:
    # Показать все кроме storage_path
else:
    # Публичный доступ - скрыть uploaded_by и storage_path
```

## Ошибки

| Код ошибки | Условие | HTTP статус | Сообщение |
|------------|---------|-------------|-----------|
| `INVALID_UUID` | Некорректный формат file_id | 400 | "Invalid file_id format" |
| `FILE_NOT_FOUND` | Файл не существует в БД | 404 | "File not found" |
| `PERMISSION_DENIED` | Недостаточно прав для просмотра деталей | 403 | "Access denied" |
| `TEXT_NOT_AVAILABLE` | include_text=true, но текст не извлечен | 409 | "Text extraction not completed yet" |
| `DATABASE_ERROR` | Ошибка выполнения запроса | 503 | "Service temporarily unavailable" |
| `STORAGE_ERROR` | Ошибка получения данных из MinIO | 503 | "Storage service error" |

## Оптимизации

### Индексы:
```sql
-- Для основного запроса
CREATE INDEX idx_files_id ON files(id);
CREATE INDEX idx_file_texts_file_id ON file_texts(file_id);

-- Для проверки прав
CREATE INDEX idx_files_uploaded_by ON files(uploaded_by) WHERE uploaded_by IS NOT NULL;
```

### Кэширование:
- Кэширование ответов по file_id
- TTL: 30 секунд для обработанных файлов, 5 секунд для pending/processing
- Инвалидация при изменении статуса обработки

### Безопасность:
- SQL injection protection через prepared statements
- Ограничение длины возвращаемого текста (configurable)
- Маскирование чувствительных данных в логах

