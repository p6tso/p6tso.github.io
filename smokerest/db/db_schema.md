```mermaid
erDiagram
    JSON_BODY ||--o{ JSON_BODY_FIELD : "body_id"
    JSON_BODY_FIELD ||--o{ JSON_BODY_FIELD : "parent_id"

    ENDPOINT ||--o{ ENDPOINT_CASE : "endpoint_id"
    ENDPOINT_CASE ||--o{ JSON_BODY : "case_id"

    JSON_BODY {
        bigint id PK
        bigint case_id FK
        enum type "request|response"
        string name
        int version
        bigint parent_body_id FK "nullable, shared schema"
        string payload "JSON строка"
        timestamp created_at
        bigint id PK
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
        int order_index
    }

    ENDPOINT {
        bigint id PK
        string path
        string description "nullable"
    }

    ENDPOINT_CASE {
        bigint id PK
        bigint endpoint_id FK
        enum method "GET|POST|PUT|PATCH|DELETE|..."
        int http_code
        string description "nullable"
    }
```