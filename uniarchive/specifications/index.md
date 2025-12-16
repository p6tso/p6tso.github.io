# Спецификации API методов

## Сервисы

### File Upload Service
**Назначение:** Прием файлов от пользователей, первичная валидация и сохранение.
**Технологии:** FastAPI, MinIO, PostgreSQL
**Основная задача:** Быстро принять файл и отправить событие в Kafka.

### Search API Service
**Назначение:** Поиск, фильтрация и предоставление файлов пользователям.
**Технологии:** FastAPI, PostgreSQL
**Основная задача:** Эффективный поиск по метаданным с пагинацией.

### Text Extraction Service
**Назначение:** Асинхронная обработка файлов, извлечение текста.
**Технологии:** Python Consumer, Kafka, библиотеки обработки файлов
**Основная задача:** Фоновая обработка без блокировки пользователя.



## Таблица методов

| Сервис | Метод                                                                | Конечная точка / Топик | Краткое описание | HTTP метод / Тип |
|--------|----------------------------------------------------------------------|------------------------|------------------|------------------|
| **File Upload Service** | [upload_file]((file-upload-service/get_upload_status.md))            | `/api/v1/upload` | Загрузка файла с метаданными (название, предмет) | POST |
| | [get_upload_status](file-upload-service/get_upload_status.md)        | `/api/v1/upload/{file_id}/status` | Проверка статуса обработки конкретного файла | GET |
| **Search API Service** | [search_files](search-api-service/search_files.md)                   | `/api/v1/files` | Поиск файлов с фильтрацией по предмету и сортировкой по дате | GET |
| | [get_file_details](search-api-service/get_file_details.md)           | `/api/v1/files/{file_id}` | Получение полной информации о файле (метаданные + статус) | GET |
| | [download_file](search-api-service/download_file.md)                 | `/api/v1/files/{file_id}/download` | Скачивание файла бинарным потоком | GET |
| | [get_file_metadata](search-api-service/get_file_metadata.md)         | `/api/v1/files/{file_id}/metadata` | Получение только технических метаданных (путь, MIME-тип, размер) | GET |
| **Text Extraction Service** | [process_file_uploaded](text-extraction-service/process_file_uploaded.md) | `file.events.uploaded` (Kafka) | Обработка события о новом файле - запуск извлечения текста | Consumer |
| | [extract_text_from_file](text-extraction-service/extract_text_from_file.md) | (внутренний) | Извлечение текста из файла (PDF, DOCX, TXT) | Функция |
| | [update_processing_status](text-extraction-service/update_processing_status.md)            | (внутренний) | Обновление статуса обработки в БД и отправка события | Функция |
