# Контракт сервиса

## Доменка эндпойнта
### post/endpoint
* получает на вход - контракт запроса ответа, чтобы ложилось на схему бд, ключ идемпотентности 
* возвращает - айдишник / ошибку
-----
запрос
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://example.com/schemas/create-endpoint-request.json",
  "title": "CreateEndpointRequest",
  "type": "object",
  "additionalProperties": false,
  "required": ["idempotencyKey", "endpoint", "requestBody", "responseBody"],
  "properties": {
    "idempotencyKey": {
      "type": "string",
      "minLength": 1,
      "maxLength": 255,
      "description": "Ключ идемпотентности для защиты от повторных созданий"
    },
    "endpoint": {
      "type": "object",
      "additionalProperties": false,
      "required": ["path"],
      "properties": {
        "path": {
          "type": "string",
          "pattern": "^/[a-zA-Z0-9_\\-/{}\\.]*$",
          "minLength": 1,
          "maxLength": 512,
          "description": "Путь эндпойнта, напр. /users/{id}"
        },
        "description": {
          "type": ["string", "null"],
          "maxLength": 1024
        }
      }
    },
    "requestBody": { "$ref": "#/definitions/jsonBody" },
    "responseBody": { "$ref": "#/definitions/jsonBody" }
  },

  "definitions": {
    "jsonBody": {
      "type": "object",
      "additionalProperties": false,
      "required": ["name", "fields"],
      "properties": {
        "name": {
          "type": "string",
          "minLength": 1,
          "maxLength": 255
        },
        "description": {
          "type": ["string", "null"],
          "maxLength": 1024
        },
        "method": {
          "type": ["string", "null"],
          "enum": ["GET", "POST", "PUT", "PATCH", "DELETE", "HEAD", "OPTIONS", null],
          "description": "Обязателен для requestBody, для responseBody может быть null"
        },
        "httpCode": {
          "type": ["integer", "null"],
          "minimum": 100,
          "maximum": 599,
          "description": "Обязателен для responseBody"
        },
        "fields": {
          "type": "array",
          "items": { "$ref": "#/definitions/field" }
        }
      },
      "allOf": [
        {
          "if": {
            "properties": { "method": { "type": "string" } },
            "required": ["method"]
          },
          "then": {
            "properties": { "httpCode": { "type": "null" } }
          }
        }
      ]
    },

    "field": {
      "type": "object",
      "additionalProperties": false,
      "required": ["name", "type", "isArray", "required"],
      "properties": {
        "name": {
          "type": "string",
          "minLength": 1,
          "maxLength": 255
        },
        "type": {
          "type": "string",
          "enum": ["string", "number", "integer", "boolean", "object", "array", "null"]
        },
        "isArray": {
          "type": "boolean",
          "default": false,
          "description": "Если true — значение является массивом указанного type"
        },
        "format": {
          "type": ["string", "null"],
          "maxLength": 64,
          "description": "напр. date-time, email, uuid"
        },
        "required": { "type": "boolean" },
        "constraints": {
          "type": ["object", "null"],
          "description": "Произвольные ограничения валидации (minLength, max, pattern и т.п.)",
          "additionalProperties": true
        },
        "children": {
          "type": "array",
          "description": "Вложенные поля (для type=object) или описание элементов (для type=array)",
          "items": { "$ref": "#/definitions/field" },
          "default": []
        }
      }
    }
  }
}
```
ответ
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://example.com/schemas/create-endpoint-response.json",
  "title": "CreateEndpointResponse",
  "type": "object",
  "additionalProperties": false,
  "required": ["endpointId"],
  "properties": {
    "endpointId": {
      "type": "integer",
      "minimum": 1
    }
  }
}
```

### get/endpoint/{id}
* получает айдишник (пользак сам заинтересован его сохранить)
* возвращает - собранный контракт либо мы его не нашли

-----
ответ
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://example.com/schemas/get-endpoint-response.json",
  "title": "GetEndpointResponse",
  "type": "object",
  "additionalProperties": false,
  "required": ["endpointId", "endpoint", "requestBody", "responseBody", "createdAt"],
  "properties": {
    "endpointId": { "type": "integer", "minimum": 1 },
    "createdAt": { "type": "string", "format": "date-time" },
    "endpoint": {
      "type": "object",
      "required": ["path"],
      "properties": {
        "path": { "type": "string" },
        "description": { "type": ["string", "null"] }
      }
    },
    "requestBody": { "$ref": "create-endpoint-request.json#/definitions/jsonBody" },
    "responseBody": { "$ref": "create-endpoint-request.json#/definitions/jsonBody" }
  }
}
```

## Доменка мок сервиса
### get/mock/{id}/{path}
* передает то, что мы готовы принять по загруженному контракту
* без квери, я хз как их обрабатывать
* проверяем что все по контракту, стреляем ошибкой
* возвращаем сгенерированный респонс по контракту

-----
ошибочный ответ
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://example.com/schemas/error.json",
  "title": "ErrorResponse",
  "type": "object",
  "additionalProperties": false,
  "required": ["code", "message"],
  "properties": {
    "code": {
      "type": "string",
      "enum": [
        "VALIDATION_ERROR",
        "NOT_FOUND",
        "IDEMPOTENCY_CONFLICT",
        "INTERNAL_ERROR"
      ]
    },
    "message": { "type": "string", "maxLength": 2048 },
    "details": {
      "type": ["object", "array", "null"],
      "description": "Опциональные детали (например, список полей с ошибками)"
    }
  }
}
```
-----

удалять эндпойнты - не хочу разрешать

