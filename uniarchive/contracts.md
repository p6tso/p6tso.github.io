# Контракты API и событий

## События Kafka

### 1. FileUploaded
Отправляется File Upload Service при успешной загрузке файла.

```json
{
  "event_id": "123e4567-e89b-12d3-a456-426614174000",
  "event_type": "FileUploaded",
  "event_version": "1.0",
  "timestamp": "2024-01-15T10:30:00Z",
  "producer": "file-upload-service",
  "data": {
    "file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "storage_path": "uniarchive/files/2024/01/a1b2c3d4.pdf",
    "original_name": "lecture_1.pdf",
    "mime_type": "application/pdf",
    "file_size": 2048576,
    "subject": "Математический анализ",
    "uploaded_by": "user_12345"
  }
}
```

### 2. TextExtracted
Отправляется Text Extraction Service после обработки файла.

```json
{
  "event_id": "223e4567-e89b-12d3-a456-426614174001",
  "event_type": "TextExtracted",
  "event_version": "1.0",
  "timestamp": "2024-01-15T10:31:12Z",
  "producer": "text-extraction-service",
  "data": {
    "file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "success": true,
    "processing_time_ms": 1250,
    "text_length": 3421,
    "extracted_pages": 15,
    "error_message": null
  }
}
```

```json
{
  "event_id": "323e4567-e89b-12d3-a456-426614174002",
  "event_type": "TextExtracted",
  "event_version": "1.0",
  "timestamp": "2024-01-15T10:35:45Z",
  "producer": "text-extraction-service",
  "data": {
    "file_id": "b2c3d4e5-f6g7-8901-bcde-f23456789012",
    "success": false,
    "processing_time_ms": 45,
    "text_length": 0,
    "extracted_pages": 0,
    "error_message": "Unsupported file format: .exe"
  }
}
```


## API контракты

### File Upload Service API

#### POST /api/v1/upload
Загрузка файла с метаданными.

**Request (multipart/form-data):**
```http
POST /api/v1/upload HTTP/1.1
Content-Type: multipart/form-data; boundary=boundary123

--boundary123
Content-Disposition: form-data; name="file"; filename="lecture.pdf"
Content-Type: application/pdf

<бинарные данные файла>
--boundary123
Content-Disposition: form-data; name="subject"
Content-Type: text/plain

Математический анализ
--boundary123
Content-Disposition: form-data; name="original_name"
Content-Type: text/plain

Лекция 1. Пределы функций
--boundary123--
```

**Response (201 Created):**
```json
{
  "file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "uploaded",
  "uploaded_at": "2024-01-15T10:30:00Z",
  "storage_url": "/api/v1/files/a1b2c3d4/download",
  "processing_status": "pending",
  "estimated_processing_time": 5
}
```

**Response (400 Bad Request):**
```json
{
  "error": "validation_error",
  "message": "Invalid file type. Supported: pdf, docx, txt",
  "details": {
    "field": "file",
    "value": "script.exe",
    "constraint": "allowed_types"
  }
}
```

#### GET /api/v1/upload/{file_id}/status
Получение статуса обработки файла.

**Response (200 OK):**
```json
{
  "file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "upload_status": "completed",
  "processing_status": "completed",
  "uploaded_at": "2024-01-15T10:30:00Z",
  "processed_at": "2024-01-15T10:31:12Z",
  "text_extracted": true,
  "text_length": 3421
}
```

### Search API Service

#### GET /api/v1/files
Поиск файлов с фильтрацией.

**Query Parameters:**
- `subject` (опционально): Фильтр по предмету
- `sort` (опционально): Сортировка (`date`, `-date`, `name`, `-name`)
- `limit` (опционально, default=50): Количество результатов
- `offset` (опционально, default=0): Смещение

**Request:**
```http
GET /api/v1/files?subject=Математический+анализ&sort=-date&limit=20 HTTP/1.1
```

**Response (200 OK):**
```json
{
  "files": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "original_name": "Лекция 1. Пределы функций",
      "subject": "Математический анализ",
      "uploaded_at": "2024-01-15T10:30:00Z",
      "file_size": 2048576,
      "mime_type": "application/pdf",
      "processing_status": "completed",
      "has_text": true,
      "download_url": "/api/v1/files/a1b2c3d4/download"
    },
    {
      "id": "b2c3d4e5-f6g7-8901-bcde-f23456789012",
      "original_name": "Практическое задание 2",
      "subject": "Математический анализ",
      "uploaded_at": "2024-01-14T15:45:30Z",
      "file_size": 104857,
      "mime_type": "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
      "processing_status": "pending",
      "has_text": false,
      "download_url": "/api/v1/files/b2c3d4e5/download"
    }
  ],
  "pagination": {
    "total": 42,
    "limit": 20,
    "offset": 0,
    "has_more": true
  },
  "filters": {
    "subject": "Математический анализ",
    "sort": "-date"
  }
}
```

#### GET /api/v1/files/{file_id}
Получение детальной информации о файле.

**Response (200 OK):**
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "original_name": "Лекция 1. Пределы функций",
  "storage_path": "uniarchive/files/2024/01/a1b2c3d4.pdf",
  "subject": "Математический анализ",
  "uploaded_at": "2024-01-15T10:30:00Z",
  "uploaded_by": "user_12345",
  "file_size": 2048576,
  "mime_type": "application/pdf",
  "processing_status": "completed",
  "processed_at": "2024-01-15T10:31:12Z",
  "text_extracted": true,
  "text_preview": "Лекция 1. Пределы функций. 1. Определение предела...",
  "text_length": 3421,
  "download_url": "/api/v1/files/a1b2c3d4/download",
  "metadata_url": "/api/v1/files/a1b2c3d4/metadata"
}
```

#### GET /api/v1/files/{file_id}/download
Скачивание файла.

**Response (200 OK):**
```
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Disposition: attachment; filename="Лекция 1. Пределы функций.pdf"
Content-Length: 2048576

<бинарные данные файла>
```

**Response (404 Not Found):**
```json
{
  "error": "not_found",
  "message": "File not found or not accessible",
  "file_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```
