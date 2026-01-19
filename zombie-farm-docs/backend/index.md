# Zombie-farm Backend

## Архитектура
* Серверная часть приложения представляет собой монолит на java
## БД

```mermaid
graph TB
    classDef frontend fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef backend fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef auth fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef db fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef external fill:#fce4ec,stroke:#880e4f,stroke-width:2px
    
    TG[Telegram API]:::external
    WS[WebSocket Server]:::external
    
    subgraph "Frontend (React + Vite + Phaser)"
        VITE[Vite Dev Server]:::frontend
        REACT[React App]:::frontend
        
        subgraph "Game Module"
            PHASER[Phaser Game Engine]:::frontend
            GAME_LOGIC[Игровая логика]:::frontend
            UI_COMPONENTS[UI Components]:::frontend
        end
        
        subgraph "Client GraphQL"
            APOLLO_CLIENT[Apollo Client]:::frontend
            GQL_QUERIES[GraphQL Queries]:::frontend
            GQL_MUTATIONS[GraphQL Mutations]:::frontend
        end
        
        subgraph "State Management"
            REDUX[Redux Store]:::frontend
            GAME_STATE[Состояние игры]:::frontend
            USER_STATE[Состояние пользователя]:::frontend
        end
    end
    
    subgraph "Backend (Spring Boot)"
        SPRING_APP[Spring Boot Application]:::backend
        
        subgraph "GraphQL Layer"
            GQL_CONTROLLER[GraphQL Controller]:::backend
            SCHEMA[GraphQL Schema]:::backend
            DATA_LOADERS[DataLoaders]:::backend
        end
        
        subgraph "Business Logic"
            GAME_SERVICE[Game Service]:::backend
            FARM_SERVICE[Farm Service]:::backend
            ZOMBIE_SERVICE[Zombie Service]:::backend
            BATTLE_SERVICE[Battle Service]:::backend
        end
        
        subgraph "Authentication"
            AUTH_CONTROLLER[Auth Controller]:::auth
            TELEGRAM_AUTH[Telegram Auth Service]:::auth
            JWT_SERVICE[JWT Service]:::auth
        end
        
        subgraph "WebSocket"
            WS_HANDLER[WebSocket Handler]:::backend
            NOTIFICATION_SERVICE[Notification Service]:::backend
        end
        
        subgraph "Persistence"
            REPOSITORIES[Spring Data Repositories]:::backend
            ENTITIES[JPA Entities]:::backend
        end
    end
    
    subgraph "Databases"
        POSTGRES[(PostgreSQL<br/>Основные данные)]:::db
    end
    
    %% Связи
    
    %% Внешние связи
    TG --> TELEGRAM_AUTH
    APOLLO_CLIENT --> WS
    
    %% Фронтенд связи
    VITE --> REACT
    REACT --> PHASER
    REACT --> APOLLO_CLIENT
    REACT --> REDUX
    PHASER --> GAME_LOGIC
    GAME_LOGIC --> APOLLO_CLIENT
    UI_COMPONENTS --> APOLLO_CLIENT
    APOLLO_CLIENT --> GQL_QUERIES
    APOLLO_CLIENT --> GQL_MUTATIONS
    
    %% Бэкенд связи
    SPRING_APP --> GQL_CONTROLLER
    SPRING_APP --> AUTH_CONTROLLER
    SPRING_APP --> WS_HANDLER
    
    GQL_CONTROLLER --> SCHEMA
    SCHEMA --> DATA_LOADERS
    DATA_LOADERS --> GAME_SERVICE
    GAME_SERVICE --> REPOSITORIES
    
    AUTH_CONTROLLER --> TELEGRAM_AUTH
    TELEGRAM_AUTH --> JWT_SERVICE
    JWT_SERVICE --> REDIS
    
    WS_HANDLER --> NOTIFICATION_SERVICE
    NOTIFICATION_SERVICE --> GAME_SERVICE
    
    %% Сервисы
    GAME_SERVICE --> FARM_SERVICE
    GAME_SERVICE --> ZOMBIE_SERVICE
    GAME_SERVICE --> BATTLE_SERVICE
    
    REPOSITORIES --> ENTITIES
    ENTITIES --> POSTGRES
    REPOSITORIES --> REDIS
    
    %% Связь фронтенд-бэкенд
    APOLLO_CLIENT -- HTTP/HTTPS --> GQL_CONTROLLER
    APOLLO_CLIENT -- WebSocket --> WS_HANDLER
    
    %% Описание потоков данных
    linkStyle 23 stroke:#ff6f00,stroke-width:2px,stroke-dasharray: 5 5
    linkStyle 24 stroke:#ff6f00,stroke-width:2px,stroke-dasharray: 5 5
```
## Интеграции
* Для интеграции в проекте использован graphql из-за своей гибкости
* Контракт

```
scalar DateTime
scalar Long
scalar JSON

type Mutation {
  buildHouse(input: BuildHouseInput!): House!
  updateHouseLevel(input: HouseIdInput!): House!
  updateHouseSkin(input: UpdateHouseSkinInput!): House!
  updateHouseLocation(input: UpdateHouseLocationInput!): House!
  removeHouse(input: HouseIdInput!): RemoveHousePayload!
  updatePlayerMeat: Player!

  convertMeatToBrain(input: ConvertMeatToBrainInput!): Player!
  convertBrainToGold(input: ConvertBrainToGoldInput!): Player!
}

type Query {
  getPlayer: Player!
  getHouse(houseId: ID!): House
  getPlayerHouses: [House!]!

  getHousesInfoCfg: JSON!
  getGameLogicCfg: JSON!
}

input BuildHouseInput {
  type: HouseType!
  skin: String!
  cell: Int!
}

input UpdateHouseSkinInput {
  houseId: ID!
  newSkin: String!
}

input UpdateHouseLocationInput {
  houseId: ID!
  newCell: Int!
}

input HouseIdInput {
  houseId: ID!
}

input ConvertMeatToBrainInput {
  meatToSpend: Long!
}

input ConvertBrainToGoldInput {
  brainToSpend: Long!
}

type RemoveHousePayload {
  success: Boolean!
  deletedHouseId: ID!
}

type Player {
  id: ID!
  username: String!
  photoUrl: String!
  meat: Long!
  gold: Long!
  brain: Long!
  boardColor: BoardColor!
  houses: [House!]!
}

type House {
  id: ID!
  playerId: Long!
  type: HouseType!
  level: Int!
  skin: String!
  cell: Int!
}

enum BoardColor {
  ORANGE
  GREEN
}

enum HouseType {
  FARM
  DECOR
  STORAGE
}
```