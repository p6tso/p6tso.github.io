# База данных UniArchive

## Модель данных

Система использует две основные сущности для хранения метаданных файлов и извлеченного текста.

### Основные таблицы

#### 1. Таблица `files` - метаданные файлов
Содержит основную информацию о загруженных файлах.

**Поля:**
- `id` (UUID, PRIMARY KEY) - уникальный идентификатор файла
- `original_name` (VARCHAR(512), NOT NULL) - оригинальное имя файла при загрузке
- `storage_path` (VARCHAR(1024), NOT NULL) - путь к файлу в объектном хранилище (MinIO/S3)
- `subject` (VARCHAR(256), NOT NULL) - учебный предмет/дисциплина
- `uploaded_at` (TIMESTAMP WITH TIME ZONE, DEFAULT NOW()) - дата и время загрузки
- `file_size` (BIGINT) - размер файла в байтах
- `mime_type` (VARCHAR(128)) - MIME-тип файла
- `uploaded_by` (VARCHAR(128)) - идентификатор пользователя, загрузившего файл

**Индексы:**
- `idx_files_subject` (subject) - для фильтрации по предмету
- `idx_files_uploaded_at` (uploaded_at DESC) - для сортировки по дате
- `idx_files_subject_date` (subject, uploaded_at DESC) - композитный индекс для частых запросов

#### 2. Таблица `file_texts` - извлеченный текст
Содержит текст, извлеченный из файлов асинхронным сервисом обработки.

**Поля:**
- `file_id` (UUID, PRIMARY KEY, FOREIGN KEY REFERENCES files(id) ON DELETE CASCADE) - ссылка на файл
- `extracted_text` (TEXT) - извлеченный текст из файла
- `processed_at` (TIMESTAMP WITH TIME ZONE, DEFAULT NOW()) - время обработки файла
- `processing_status` (VARCHAR(32), DEFAULT 'pending') - статус обработки: pending/processing/completed/failed
- `error_message` (TEXT) - сообщение об ошибке при обработке

**Индексы:**
- `idx_file_texts_status` (processing_status) - для поиска необработанных файлов

### PlantUML код для ER-диаграммы

```mermaid
@startuml UniArchive_ER_Diagram

!define primary_key #aliceblue
!define foreign_key #lightcoral

entity "files" {
  *id : uuid <<PK>>
  --
  *original_name : varchar(512)
  *storage_path : varchar(1024)
  *subject : varchar(256)
  *uploaded_at : timestamp
  file_size : bigint
  mime_type : varchar(128)
  uploaded_by : varchar(128)
}

entity "file_texts" {
  *file_id : uuid <<PK>> <<FK>>
  --
  extracted_text : text
  processed_at : timestamp
  processing_status : varchar(32)
  error_message : text
}

files ||..o{ file_texts

' Добавляем индексы
note bottom of files
  Индексы:
  - idx_files_subject (subject)
  - idx_files_uploaded_at (uploaded_at)
  - idx_files_subject_date (subject, uploaded_at)
end note

note bottom of file_texts
  Индекс:
  - idx_file_texts_status (processing_status)
end note

```




## Схема взаимодействия с сервисами

### 1. File Upload Service
```sql
-- Вставляет запись при загрузке файла
INSERT INTO files (id, original_name, storage_path, subject, file_size, mime_type, uploaded_by)
VALUES (:id, :original_name, :storage_path, :subject, :file_size, :mime_type, :uploaded_by);

-- Создает запись-заглушку для асинхронной обработки
INSERT INTO file_texts (file_id, processing_status) VALUES (:file_id, 'pending');
```

### 2. Text Extraction Service
```sql
-- Находит файлы для обработки
SELECT f.id, f.storage_path, f.mime_type 
FROM files f
JOIN file_texts ft ON f.id = ft.file_id
WHERE ft.processing_status = 'pending'
LIMIT 10;

-- Обновляет статус на "в обработке"
UPDATE file_texts 
SET processing_status = 'processing', processed_at = NOW()
WHERE file_id = :file_id;

-- Сохраняет извлеченный текст
UPDATE file_texts 
SET extracted_text = :text, 
    processing_status = 'completed',
    processed_at = NOW()
WHERE file_id = :file_id;

-- Обработка ошибки
UPDATE file_texts 
SET processing_status = 'failed',
    error_message = :error_message,
    processed_at = NOW()
WHERE file_id = :file_id;
```

### 3. Search API Service
```sql
-- Поиск файлов с фильтрацией по предмету и сортировкой
SELECT f.id, f.original_name, f.subject, f.uploaded_at, f.file_size,
       ft.processing_status, ft.processed_at
FROM files f
LEFT JOIN file_texts ft ON f.id = ft.file_id
WHERE f.subject = :subject  -- фильтр по предмету
ORDER BY f.uploaded_at DESC  -- сортировка по дате
LIMIT :limit OFFSET :offset;

-- Получение полной информации о файле
SELECT f.*, ft.extracted_text, ft.processing_status
FROM files f
LEFT JOIN file_texts ft ON f.id = ft.file_id
WHERE f.id = :file_id;
```

## Миграции базы данных

### Версия 1.0 (Initial Schema)
```sql
-- Создание таблицы files
CREATE TABLE files (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    original_name VARCHAR(512) NOT NULL,
    storage_path VARCHAR(1024) NOT NULL,
    subject VARCHAR(256) NOT NULL,
    uploaded_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    file_size BIGINT,
    mime_type VARCHAR(128),
    uploaded_by VARCHAR(128)
);

-- Создание таблицы file_texts
CREATE TABLE file_texts (
    file_id UUID PRIMARY KEY REFERENCES files(id) ON DELETE CASCADE,
    extracted_text TEXT,
    processed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    processing_status VARCHAR(32) DEFAULT 'pending',
    error_message TEXT
);

-- Создание индексов
CREATE INDEX idx_files_subject ON files(subject);
CREATE INDEX idx_files_uploaded_at ON files(uploaded_at DESC);
CREATE INDEX idx_files_subject_date ON files(subject, uploaded_at DESC);
CREATE INDEX idx_file_texts_status ON file_texts(processing_status);
```

## Оптимизации и масштабирование

### 1. Партиционирование (для больших объемов)
```sql
-- Партиционирование таблицы files по предмету
CREATE TABLE files_partitioned (
    -- те же поля что и в files
) PARTITION BY LIST (subject);
```

### 2. Полнотекстовый поиск
```sql
-- Добавление GIN индекса для поиска по тексту
CREATE INDEX idx_file_texts_gin ON file_texts 
USING gin(to_tsvector('russian', extracted_text));

-- Поиск по извлеченному тексту
SELECT f.* 
FROM files f
JOIN file_texts ft ON f.id = ft.file_id
WHERE ft.processing_status = 'completed'
AND to_tsvector('russian', ft.extracted_text) @@ to_tsquery('russian', 'линейная & алгебра');
```

### 3. Статистика и мониторинг
```sql
-- Статистика по предметам
SELECT subject, COUNT(*) as file_count, 
       SUM(file_size) as total_size,
       AVG(file_size) as avg_size
FROM files 
GROUP BY subject 
ORDER BY file_count DESC;

-- Статистика обработки
SELECT processing_status, COUNT(*) 
FROM file_texts 
GROUP BY processing_status;
```

## Резервное копирование и восстановление

### Ежедневный бэкап метаданных
```bash
pg_dump -U postgres -d uniarchive -t files -t file_texts > backup_$(date +%Y%m%d).sql
```

### Восстановление
```sql
-- Восстановление схемы и данных
psql -U postgres -d uniarchive < backup_file.sql
```

## Безопасность

### 1. Роли и привилегии
```sql
-- Создание роли для сервисов
CREATE ROLE uniarchive_service LOGIN PASSWORD 'secure_password';

-- Выдача минимально необходимых прав
GRANT SELECT, INSERT, UPDATE ON files, file_texts TO uniarchive_service;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO uniarchive_service;

-- Создание read-only роли для аналитики
CREATE ROLE uniarchive_readonly;
GRANT SELECT ON files, file_texts TO uniarchive_readonly;
```

### 2. Аудит изменений
```sql
-- Таблица для аудита изменений
CREATE TABLE files_audit (
    id SERIAL PRIMARY KEY,
    file_id UUID REFERENCES files(id),
    changed_at TIMESTAMP DEFAULT NOW(),
    changed_by VARCHAR(128),
    operation VARCHAR(10),
    old_data JSONB,
    new_data JSONB
);

-- Триггер для отслеживания изменений
CREATE OR REPLACE FUNCTION audit_files_change() 
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO files_audit (file_id, changed_by, operation, old_data, new_data)
    VALUES (
        COALESCE(NEW.id, OLD.id),
        current_user,
        TG_OP,
        CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN row_to_json(OLD) END,
        CASE WHEN TG_OP IN ('INSERT', 'UPDATE') THEN row_to_json(NEW) END
    );
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER files_audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON files
FOR EACH ROW EXECUTE FUNCTION audit_files_change();
```

Эта схема обеспечивает надежное хранение метаданных, поддерживает асинхронную обработку через статусы и оптимизирована для частых операций поиска по предмету и дате.