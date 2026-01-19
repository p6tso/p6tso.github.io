# Zombie-farm Backend

## Архитектура
* Серверная часть приложения представляет собой монолит на java
## БД
постгря сука как сюда юмл вставить бляди
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