# get_upload_status

## 1. Валидация входных параметров

| Операция | Источник данных | Тип данных | Обязательность | Проверка |
|----------|-----------------|------------|----------------|----------|
| Получение file_id | Path parameter | String | Обязательно | Формат UUID v4 |
| Извлечение uploaded_by | Контекст запроса (JWT/сессия) | String | Опционально | Для проверки прав доступа |

## 2. Проверка существования файла

**Последовательные операции:**
1. Преобразование строки file_id в объект UUID
2. Подготовка SQL запроса к таблице `files`
3. Добавление условия проверки прав (если uploaded_by не NULL)
4. Выполнение запроса с таймаутом 3 секунды

**SQL запрос:**
```sql
SELECT f.id, f.original_name, f.subject, f.uploaded_at, f.uploaded_by,
       ft.processing_status, ft.processed_at, ft.error_message
FROM files f
LEFT JOIN file_texts ft ON f.id = ft.file_id
WHERE f.id = :file_id
  AND (:uploaded_by IS NULL OR f.uploaded_by = :uploaded_by)
LIMIT 1
```

## 3. Обработка результата запроса

**Логика обработки:**
1. Если запрос вернул 0 строк → файл не найден
2. Если uploaded_by указан и не совпадает → недостаточно прав
3. Извлечение данных из результата:
    - Основные метаданные из таблицы `files`
    - Статус обработки из таблицы `file_texts`
    - Время обработки и ошибки (если есть)

**Определение общего статуса:**
| Условие | Статус | Описание |
|---------|--------|----------|
| `file_texts` запись отсутствует | `unknown` | Файл есть, но статус не определен |
| `processing_status = 'pending'` | `pending` | Ожидает обработки |
| `processing_status = 'processing'` | `processing` | В процессе извлечения текста |
| `processing_status = 'completed'` | `completed` | Текст успешно извлечен |
| `processing_status = 'failed'` | `failed` | Ошибка при обработке |

## 4. Расчет дополнительных метрик

**Последовательные операции:**
1. Определение времени в очереди (если статус `pending`):
    - `queue_time = current_time - uploaded_at`
2. Определение времени обработки (если статус `completed` или `failed`):
    - `processing_time = processed_at - uploaded_at`
3. Проверка наличия события в Kafka (опционально):
    - Запрос к топику `file.events.uploaded` по ключу file_id
    - Проверка, было ли событие обработано

## 5. Формирование ответа

**Последовательные операции:**
1. Установка HTTP статуса: `200 OK` (при успехе)
2. Подготовка заголовков:
    - `Content-Type: application/json`
    - `X-File-Status: {status}` (для упрощения клиентской логики)
3. Сбор данных в структуру ответа

**Тело ответа (успех):**
```json
{
  "file_id": "uuid",
  "status": "uploaded|processing|completed|failed|unknown",
  "upload_details": {
    "original_name": "string",
    "subject": "string",
    "uploaded_at": "ISO 8601",
    "uploaded_by": "string или null",
    "file_size": 123456
  },
  "processing_details": {
    "status": "pending|processing|completed|failed",
    "started_at": "ISO 8601 или null",
    "completed_at": "ISO 8601 или null",
    "error_message": "string или null",
    "queue_time_seconds": 123,
    "processing_time_seconds": 45
  },
  "links": {
    "download": "/api/v1/files/{file_id}/download",
    "details": "/api/v1/files/{file_id}",
    "metadata": "/api/v1/files/{file_id}/metadata"
  },
  "timestamps": {
    "current_time": "ISO 8601",
    "next_status_check": "ISO 8601 (рекомендуемое время следующей проверки)"
  }
}
```

## 6. Кэширование (опционально, для производительности)

**Логика кэширования:**
1. Ключ кэша: `file_status:{file_id}`
2. TTL кэша:
    - `pending`: 10 секунд
    - `processing`: 5 секунд
    - `completed`: 60 секунд
    - `failed`: 30 секунд
3. Инвалидация кэша при изменении статуса

## Ошибки

| Код ошибки | Условие | HTTP статус | Сообщение | Действие |
|------------|---------|-------------|-----------|----------|
| `INVALID_UUID` | Некорректный формат file_id | 400 | "Invalid file_id format. Expected UUID v4" | Возврат ошибки без запроса к БД |
| `FILE_NOT_FOUND` | Файл не существует в БД | 404 | "File not found with id: {file_id}" | Логирование попытки доступа |
| `PERMISSION_DENIED` | uploaded_by не совпадает с владельцем | 403 | "You don't have permission to view this file status" | Без уточнения существования файла |
| `DATABASE_ERROR` | Ошибка подключения/запроса к БД | 503 | "Service temporarily unavailable" | Ретрай через 2 секунды |
| `TIMEOUT_ERROR` | Таймаут запроса к БД | 504 | "Request timeout" | Увеличить timeout или оптимизировать запрос |
