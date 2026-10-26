# БД

## Схема

```mermaid
erDiagram
    JSON_BODY {
        bigint id PK
        bigint method_id FK
        enum type "request|response"
        enum method "nullable, GET|POST|PUT|PATCH|DELETE|..."
        int http_code "nullable"
        string description "nullable"
        string name
        string payload "JSON строка"
        timestamp created_at
    }

    JSON_BODY_FIELD {
        bigint id PK
        bigint body_id FK
        bigint parent_id FK "nullable, вложенность"
        string name
        enum type "string|number|integer|boolean|object|array|null"
        boolean is_array
        string format "nullable"
        boolean required
        string constraints "nullable, JSON строка"
    }

    METHOD {
        bigint id PK
        uuid endpoint_id FK 
        bigint current_response_id FK
        string path
        string description "nullable"
    }
    
    ENDPOINT {
        uuid id PK
        string name
    }
    JSON_BODY ||--}{ JSON_BODY_FIELD : "body_id"
    JSON_BODY_FIELD ||--|o JSON_BODY_FIELD : "parent_id"

    METHOD ||--{ JSON_BODY : "method_id"
    JSON_BODY ||--|| METHOD : "body_id"
    
    ENDPOINT ||--}{ METHOD : "endpoint_id"
```