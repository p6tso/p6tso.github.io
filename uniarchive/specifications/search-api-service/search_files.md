# search_files

## 1. Валидация query параметров

| Параметр | Источник | Тип данных | Обязательность | Допустимые значения | По умолчанию |
|----------|----------|------------|----------------|---------------------|--------------|
| `subject` | Query string | String | Опционально | Длина 1-256 символов | NULL (все предметы) |
| `sort` | Query string | String | Опционально | `date`, `-date`, `name`, `-name` | `-date` |
| `limit` | Query string | Integer | Опционально | 1-100 | 50 |
| `offset` | Query string | Integer | Опционально | ≥ 0 | 0 |
| `status` | Query string | String | Опционально | `all`, `processed`, `unprocessed` | `all` |

## 2. Построение SQL запроса

**Последовательные операции:**
1. Базовый запрос с JOIN таблиц `files` и `file_texts`
2. Динамическое добавление WHERE условий на основе параметров
3. Применение сортировки в зависимости от параметра `sort`
4. Добавление LIMIT и OFFSET для пагинации
5. Подготовка второго запроса для подсчета общего количества

**Базовый SQL запрос:**
```sql
SELECT f.id, f.original_name, f.subject, f.uploaded_at, 
       f.file_size, f.mime_type, f.uploaded_by,
       ft.processing_status, ft.processed_at
FROM files f
LEFT JOIN file_texts ft ON f.id = ft.file_id
WHERE 1=1
```

## 3. Применение фильтров

### Фильтр по предмету (subject):
```sql
AND f.subject = :subject
```

### Фильтр по статусу обработки:
| Значение `status` | SQL условие |
|-------------------|-------------|
| `processed` | `AND ft.processing_status = 'completed'` |
| `unprocessed` | `AND (ft.processing_status IS NULL OR ft.processing_status IN ('pending', 'failed'))` |
| `all` | Без условия |

### Фильтр по пользователю (если требуется):
```sql
AND f.uploaded_by = :current_user_id
```

## 4. Применение сортировки

| Значение `sort` | SQL ORDER BY |
|-----------------|--------------|
| `date` | `f.uploaded_at ASC` |
| `-date` | `f.uploaded_at DESC` |
| `name` | `f.original_name ASC` |
| `-name` | `f.original_name DESC` |

## 5. Пагинация и подсчет

**Последовательные операции:**
1. Применение лимита и оффсета к основному запросу:
   ```sql
   LIMIT :limit OFFSET :offset
   ```
2. Выполнение запроса для получения данных
3. Подготовка и выполнение запроса для подсчета общего количества:
   ```sql
   SELECT COUNT(*) as total_count FROM files f ...
   ```
4. Расчет метаданных пагинации:
    - `has_next_page = (offset + limit) < total_count`
    - `has_previous_page = offset > 0`
    - `total_pages = ceil(total_count / limit)`

## 6. Форматирование результатов

**Преобразование каждой строки результата:**
```json
{
  "id": "uuid из БД",
  "original_name": "строка из БД",
  "subject": "строка из БД",
  "uploaded_at": "ISO 8601 timestamp",
  "file_size": "число в байтах",
  "mime_type": "строка MIME-типа",
  "processing_status": "строка статуса или null",
  "processed_at": "ISO 8601 или null",
  "links": {
    "self": "/api/v1/files/{id}",
    "download": "/api/v1/files/{id}/download",
    "status": "/api/v1/upload/{id}/status"
  }
}
```

## 7. Формирование ответа

**Тело ответа:**
```json
{
  "files": [
    // массив отформатированных файлов
  ],
  "pagination": {
    "total": 1234,
    "limit": 50,
    "offset": 0,
    "has_next": true,
    "has_previous": false,
    "total_pages": 25,
    "current_page": 1
  },
  "filters": {
    "applied": {
      "subject": "Математика",
      "status": "processed",
      "sort": "-date"
    },
    "available": {
      "subjects": ["Математика", "Физика", "Программирование"],
      "statuses": ["all", "processed", "unprocessed"],
      "sorts": ["date", "-date", "name", "-name"]
    }
  },
  "metadata": {
    "query_time_ms": 45,
    "returned_count": 50,
    "timestamp": "ISO 8601"
  }
}
```

## Ошибки

| Код ошибки | Условие | HTTP статус | Сообщение |
|------------|---------|-------------|-----------|
| `INVALID_LIMIT` | limit < 1 или limit > 100 | 400 | "Limit must be between 1 and 100" |
| `INVALID_OFFSET` | offset < 0 | 400 | "Offset cannot be negative" |
| `INVALID_SORT` | sort не из допустимых значений | 400 | "Invalid sort parameter. Allowed: date, -date, name, -name" |
| `INVALID_STATUS` | status не из допустимых значений | 400 | "Invalid status filter. Allowed: all, processed, unprocessed" |
| `DATABASE_ERROR` | Ошибка выполнения запроса | 503 | "Search service temporarily unavailable" |
| `TIMEOUT_ERROR` | Таймаут запроса к БД | 504 | "Search request timeout" |

## Оптимизации

### Индексы для ускорения поиска:
```sql
-- Для фильтрации по subject + сортировки по дате
CREATE INDEX idx_files_subject_date ON files(subject, uploaded_at DESC);

-- Для фильтрации по статусу обработки
CREATE INDEX idx_file_texts_status ON file_texts(processing_status);
```

### Кэширование:
- Кэширование популярных запросов (subject + sort комбинации)
- TTL кэша: 30 секунд для динамичных данных
- Инвалидация при добавлении новых файлов

### Безопасность:
- Максимальный лимит записей: 100
- SQL injection protection через parameterized queries
- Rate limiting: 100 запросов в минуту на IP