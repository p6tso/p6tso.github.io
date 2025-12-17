# download_file

## 1. Валидация входных параметров

| Параметр | Источник | Тип данных | Обязательность | Проверка |
|----------|----------|------------|----------------|----------|
| `file_id` | Path parameter | String | Обязательно | Формат UUID v4 |
| `download_as` | Query string | String | Опционально | Имя файла для Content-Disposition |
| `preview` | Query string | Boolean | Опционально | true/false, по умолчанию false |

## 2. Проверка существования файла и прав доступа

**Последовательные операции:**
1. Запрос к таблице `files` для получения:
    - `storage_path` (путь в MinIO)
    - `original_name` (для имени файла по умолчанию)
    - `mime_type` (для Content-Type)

```sql
SELECT f.storage_path, f.original_name, f.mime_type, f.uploaded_by,
       f.file_size, ft.processing_status
FROM files f
LEFT JOIN file_texts ft ON f.id = ft.file_id
WHERE f.id = :file_id
```

## 3. Генерация ссылки для скачивания из MinIO

**Типы ссылок:**
1. **Прямая переадресация** (рекомендуется):
    - Генерация presigned URL с TTL 5 минут
    - HTTP 302 Redirect на этот URL
    - Клиент скачивает напрямую из MinIO

2. **Проксирование через сервис**:
    - Stream файла из MinIO через сервис
    - Полный контроль над отдачей
    - Большая нагрузка на сервис

### Presigned URL генерация:
```python
presigned_url = minio_client.presigned_get_object(
    bucket_name="uniarchive-files",
    object_name=storage_path,
    expires=timedelta(minutes=5)
)
```

## 4. Настройка заголовков ответа

### Для прямого скачивания (redirect):
```
HTTP/1.1 302 Found
Location: https://minio.example.com/uniarchive-files/2024/01/uuid.pdf?X-Amz-...
X-File-ID: uuid
X-File-Size: 1234567
X-Original-Name: Лекция 1.pdf (URL-encoded)
Cache-Control: private, max-age=300
```

### Для проксирования (stream):
```
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Disposition: attachment; filename="Лекция 1.pdf"; filename*=UTF-8''%D0%9B%D0%B5%D0%BA%D1%86%D0%B8%D1%8F%201.pdf
Content-Length: 1234567
Content-Transfer-Encoding: binary
Accept-Ranges: bytes
Last-Modified: Wed, 15 Jan 2024 10:30:00 GMT
ETag: "abc123def456"
Cache-Control: private, max-age=3600
X-File-ID: uuid
X-Processing-Status: completed
```

## 5. Обработка параметра `preview`

**Если `preview=true`:**
1. Проверка MIME-типа:
    - PDF → отдача как есть (браузер покажет preview)
    - Другие типы → обычное скачивание
2. Изменение Content-Disposition:
   ```http
   Content-Disposition: inline; filename="document.pdf"
   ```
3. Для изображений → добавление заголовков кэширования

## 6. Логика именования файла

**Приоритет именования:**
1. Параметр `download_as` (если передан)
2. `original_name` из БД
3. `file_id` + расширение из MIME-типа

**Кодирование имени:**
- RFC 5987 для Unicode имён
- Fallback на ASCII для старых клиентов
```http
filename="simple.pdf"
filename*=UTF-8''%D0%A1%D0%BB%D0%BE%D0%B6%D0%BD%D0%BE%D0%B5%20%D0%B8%D0%BC%D1%8F.pdf
```

## 7. Формирование ответа

### Успешный ответ (проксирование):
```
HTTP/1.1 200 OK
Content-Type: {mime_type}
Content-Disposition: {attachment/inline}; filename="..."
Content-Length: {file_size}

<stream файла из MinIO>
```

### Redirect ответ:
```
HTTP/1.1 302 Found
Location: {presigned_url}
X-File-Metadata: {"id":"uuid","name":"filename","size":1234567}
```

## Ошибки

| Код ошибки | Условие | HTTP статус | Сообщение | Действие |
|------------|---------|-------------|-----------|----------|
| `INVALID_UUID` | Некорректный file_id | 400 | "Invalid file identifier" | - |
| `FILE_NOT_FOUND` | Файл не в БД | 404 | "File not found" | Логирование попытки |
| `STORAGE_NOT_FOUND` | Файл в БД, но не в MinIO | 410 | "File no longer available" | Пометить как удалённый |
| `PERMISSION_DENIED` | Нет прав на скачивание | 403 | "Access denied" | - |
| `STORAGE_UNAVAILABLE` | MinIO недоступен | 503 | "Storage service unavailable" | Ретрай через 30 сек |
| `FILE_TOO_LARGE` | Файл > 1GB при проксировании | 413 | "File too large for direct download" | Всегда redirect |
| `DOWNLOAD_LIMIT_EXCEEDED` | Превышен лимит скачиваний | 429 | "Download limit exceeded" | Retry-After: 3600 |

## Оптимизации

### Кэширование presigned URL:
- Ключ: `presigned:{file_id}:{expires_in}`
- TTL: на 1 минуту меньше чем срок действия URL
- Инвалидация: при изменении файла в MinIO

### Rate limiting:
- 100 скачиваний в час на пользователя
- 1000 скачиваний в час на файл
- Мониторинг подозрительной активности

### Частичное скачивание (Range requests):
```http
GET /api/v1/files/{id}/download
Range: bytes=0-999
```
- Поддержка заголовка `Range`
- Правильные ответы `206 Partial Content`
- Заголовок `Accept-Ranges: bytes`

