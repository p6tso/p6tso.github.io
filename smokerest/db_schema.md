# БД

## Схема

```mermaid

erDiagram
    JSON_BODY {
        bigint id PK
        bigint method_id FK
        enum type "request | response"
        enum method "nullable, HTTP method"
        int http_code "nullable"
        varchar(512) description "nullable"
        varchar(64) name
        json payload "JSON string"
        timestamp created_at
    }

    JSON_BODY_FIELD {
        bigint id PK
        bigint body_id FK
        bigint parent_id FK "nullable, nested field"
        varchar(64) name
        enum type "string, number, integer, boolean, object, array, null"
        boolean is_array
        varchar(256) format "nullable"
        boolean required
    }

    METHOD {
        bigint id PK
        uuid endpoint_id FK
        bigint current_response_id FK
        varchar(128) path
        varchar(512) description "nullable"
    }

    ENDPOINT {
        uuid id PK
        varchar(64) name
    }

    METHOD ||--o{ JSON_BODY : "method_id"
    JSON_BODY ||--o{ JSON_BODY_FIELD : "body_id"
    JSON_BODY_FIELD o|--o{ JSON_BODY_FIELD : "parent_id"
    ENDPOINT ||--o{ METHOD : "endpoint_id"
    METHOD o|--o| JSON_BODY : "current_response_id"

```