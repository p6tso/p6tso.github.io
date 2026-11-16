# Контракт сервиса

## Frontend → Backend

POST `/api/endpoints`

Request:

```json
{
  "name": "Users API",
  "methods": [
    {
      "path": "/users",
      "description": "Users endpoint",
      "method": "POST",
      "request": {
        "name": "Create user",
        "description": "User creation request",
        "payload": {
          "name": "John",
          "age": 25,
          "address": {
            "city": "Berlin"
          }
        },
        "fields": [
          {
            "name": "name",
            "type": "string",
            "isArray": false,
            "format": null,
            "required": true,
            "children": []
          },
          {
            "name": "age",
            "type": "integer",
            "isArray": false,
            "format": null,
            "required": true,
            "children": []
          },
          {
            "name": "address",
            "type": "object",
            "isArray": false,
            "format": null,
            "required": true,
            "children": [
              {
                "name": "city",
                "type": "string",
                "isArray": false,
                "format": null,
                "required": true,
                "children": []
              }
            ]
          }
        ]
      },
      "responses": [
        {
          "name": "Success",
          "httpCode": 201,
          "payload": {
            "id": 1,
            "name": "John"
          },
          "fields": [
            {
              "name": "id",
              "type": "integer",
              "isArray": false,
              "format": null,
              "required": true,
              "children": []
            },
            {
              "name": "name",
              "type": "string",
              "isArray": false,
              "format": null,
              "required": true,
              "children": []
            }
          ]
        },
        {
          "name": "Bad request",
          "httpCode": 400,
          "payload": {
            "error": "Invalid data"
          },
          "fields": [
            {
              "name": "error",
              "type": "string",
              "isArray": false,
              "format": null,
              "required": true,
              "children": []
            }
          ]
        }
      ]
    }
  ]
}
```

Response:

```
{
  "id": "uuid",
  "name": "Users API"
}
```

GET `/api/endpoints`

Response:

```
[
  {
    "id": "uuid",
    "name": "Users API"
  }
]
```

GET `/api/endpoints/{endpointId}`

Response:

```json
{
  "id": "uuid",
  "name": "Users API",
  "methods": [
    {
      "id": 1,
      "path": "/users",
      "method": "POST",
      "description": "Users endpoint",
      "request": {
        "id": 10,
        "name": "Create user",
        "description": "User creation request",
        "payload": {
          "name": "John",
          "age": 25
        },
        "fields": [
          {
            "id": 1,
            "name": "name",
            "type": "string",
            "isArray": false,
            "format": null,
            "required": true,
            "children": []
          },
          {
            "id": 2,
            "name": "age",
            "type": "integer",
            "isArray": false,
            "format": null,
            "required": true,
            "children": []
          }
        ]
      },
      "responses": [
        {
          "id": 11,
          "name": "Success",
          "httpCode": 201,
          "payload": {
            "id": 1,
            "name": "John"
          },
          "fields": [
            {
              "id": 3,
              "name": "id",
              "type": "integer",
              "isArray": false,
              "format": null,
              "required": true,
              "children": []
            },
            {
              "id": 4,
              "name": "name",
              "type": "string",
              "isArray": false,
              "format": null,
              "required": true,
              "children": []
            }
          ],
          "isActive": true
        }
      ]
    }
  ]
}
```

PUT `/api/methods/{methodId}/current-response`

Request:

```
{
  "responseId": 12
}
```

Response:

```
{
  "success": true,
  "currentResponseId": 12
}
```

## Backend → Backend (Mock API)

GET `/mock/{endpointId}/{path}`

POST `/mock/{endpointId}/{path}`

PUT `/mock/{endpointId}/{path}`

PATCH `/mock/{endpointId}/{path}`

DELETE `/mock/{endpointId}/{path}`

Request (пример):

```
{
  "name": "John",
  "age": 25
}
```

Response (пример):

```
{
  "id": 1,
  "name": "John",
  "age": 25
}
```

HTTP Status: `201 Created`

