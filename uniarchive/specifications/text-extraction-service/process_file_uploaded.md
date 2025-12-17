# process_file_uploaded

## 1. Валидация и парсинг события Kafka

| Поле события | Тип данных | Обязательность | Проверка |
|--------------|------------|----------------|----------|
| `event_id` | String | Обязательно | Формат UUID |
| `event_type` | String | Обязательно | Значение: "FileUploaded" |
| `timestamp` | String | Обязательно | Валидная дата ISO 8601, не в будущем |
| `data.file_id` | String | Обязательно | Формат UUID |
| `data.storage_path` | String | Обязательно | Формат пути MinIO |
| `data.mime_type` | String | Обязательно | Поддерживаемый тип (pdf, docx, txt) |
| `data.original_name` | String | Обязательно | Не пустая строка |
| `data.file_size` | Integer | Обязательно | > 0 и < 200MB |
| `data.subject` | String | Обязательно | Не пустая строка |

## 2. Обновление статуса обработки в БД

**Последовательные операции:**
1. Проверка существования файла и текущего статуса:
```sql
SELECT processing_status FROM file_texts WHERE file_id = :file_id
```
2. Если статус уже `processing` или `completed` → пропустить обработку
3. Обновление статуса на `processing`:
```sql
UPDATE file_texts 
SET processing_status = 'processing',
    processed_at = NOW(),
    error_message = NULL
WHERE file_id = :file_id
```


## 3. Загрузка файла из MinIO

**Логика загрузки:**
1. Извлечение bucket и object key из `storage_path`
2. Проверка существования файла в MinIO
3. Загрузка файла во временную директорию:
    - Путь: `/tmp/uniarchive/{file_id}/{timestamp}/original.{ext}`
    - Создание уникального имени для избежания коллизий
4. Верификация размера файла (совпадение с `file_size` из события)

## 4. Определение процессора по MIME-типу

| MIME-тип | Процессор | Метод обработки | Поддерживаемые расширения |
|----------|-----------|-----------------|---------------------------|
| `application/pdf` | PDFProcessor | `extract_from_pdf()` | .pdf |
| `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | DOCXProcessor | `extract_from_docx()` | .docx, .docm |
| `text/plain` | TextProcessor | `extract_from_text()` | .txt, .md, .csv |
| `unsupported` | - | - | Все остальные → ошибка |

## 5. Извлечение текста

- Установить статус файла - `processing`
```sql
UPDATE file_texts
SET extracted_text = :text_content,
processing_status = 'processing',
processed_at = NOW(),
text_length = LENGTH(:text_content),
error_message = NULL
WHERE file_id = :file_id
```
- **Вызвать метод [extract_text_from_file](extract_text_from_file.md)**


## 6. Обновление результата в БД

**Последовательные операции:**
1. Если извлечение успешно:
```sql
UPDATE file_texts 
SET extracted_text = :text_content,
    processing_status = 'completed',
    processed_at = NOW(),
    text_length = LENGTH(:text_content),
    error_message = NULL
WHERE file_id = :file_id
```
2. Если ошибка извлечения:
```sql
UPDATE file_texts 
SET processing_status = 'failed',
    processed_at = NOW(),
    error_message = :error_details,
    extracted_text = NULL
WHERE file_id = :file_id
```


## 8. Очистка временных файлов

**Последовательные операции:**
1. Удаление временного файла из `/tmp/`
2. Очистка пустых директорий
3. Проверка использования дискового пространства
4. При превышении лимита → принудительная очистка старых файлов

## Ошибки обработки

| Код ошибки | Условие | Действие | Статус в БД |
|------------|---------|----------|-------------|
| `INVALID_EVENT_FORMAT` | Невалидный JSON или отсутствуют обязательные поля | Пропустить событие, логировать | Не меняется |
| `FILE_NOT_IN_DB` | file_id нет в таблице files | Пропустить, отправить alert | Не меняется |
| `ALREADY_PROCESSED` | Статус уже `completed` или `processing` | Пропустить, логировать | Не меняется |
| `STORAGE_FILE_NOT_FOUND` | Файл отсутствует в MinIO | `failed`, логировать | `failed` |
| `UNSUPPORTED_FILE_TYPE` | Неподдерживаемый MIME-тип | `failed`, логировать | `failed` |
| `EXTRACTION_ERROR` | Ошибка при извлечении текста | `failed`, сохранить ошибку | `failed` |
| `DATABASE_ERROR` | Ошибка обновления БД | Retry через 30 секунд | Зависит от транзакции |
| `KAFKA_ERROR` | Ошибка отправки события | Retry отправки, но статус в БД обновлен | Обновлен |

