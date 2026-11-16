# Home Storage App 技术方案与架构设计

## 1. 项目目标

Home Storage App 是一个本地优先的家庭物品收纳管理应用。用户可以建立房间和收纳位置，记录物品及数量，通过照片和位置层级快速找到物品，并为后续的云端同步、AI 查询和简易 3D 展示预留扩展能力。

第一版重点验证以下闭环：

1. 创建房间。
2. 创建房间内的位置层级。
3. 为位置拍照或选择本地图片。
4. 添加物品、数量和备注。
5. 根据关键词查找物品。
6. 查看物品所在的房间和具体位置。
7. 修改数量、移动位置和删除物品。
8. 在房间照片上添加可点击的位置标记。

第一版暂不追求精确的自动 3D 建模、多人共享和全自动物品识别。

## 2. 总体架构

```text
页面层
  ↓
应用服务层
  ↓
仓储接口层 ───── 图片资源接口层 ───── 搜索接口层
  ↓                    ↓                    ↓
本地元数据存储       本地文件存储          本地关键词搜索

未来可替换为：云数据库、云对象存储、AI 意图识别服务
```

核心原则：

- 页面层不直接调用本地存储 API。
- 结构化数据和图片文件分开保存。
- 房间位置使用树形数据表达。
- 3D 模型属于展示层，不成为核心业务数据的依赖。
- 通过接口隔离本地实现和未来云端实现。

## 3. 本地存储设计

### 3.1 结构化数据

第一版使用本地仓储实现保存房间、位置、物品、图片引用和操作记录。数据从一开始按独立实体组织，不把全部内容保存为单个页面对象。

建议的数据集合：

```text
homes
rooms
locations
items
photos
inventory_events
app_meta
```

本地实现可以先使用 JSON 序列化保存；当数据规模增加时，可以替换为 SQLite 或其他本地数据库实现，应用服务层无需改变。

### 3.2 图片和模型文件

图片不直接以 Base64 保存到 JSON 中。文件和元数据分开管理：

```text
photos/
  room/
  location/
  item/
models/
thumbnails/
```

数据库只保存文件引用、文件类型、所属实体和缩略图路径。将来迁移到云端时，可把本地路径替换成对象存储 URL。

## 4. 核心数据模型

### Home

```text
id
name
createdAt
updatedAt
```

### Room

```text
id
homeId
name
coverPhotoId
sortOrder
createdAt
updatedAt
```

### Location

位置通过 `parentId` 支持多级层次：

```text
id
roomId
parentId
name
type
photoId
position
description
createdAt
updatedAt
```

位置类型建议包括：

```text
room_area
cabinet
drawer
shelf
box
bag
other
```

示例层级：

```text
卧室
└── 衣柜
    └── 左侧柜门
        └── 第二层
            └── 蓝色收纳盒
```

`position` 第一版保存照片上的二维坐标，未来可以扩展为 3D 变换信息。

### Item

```text
id
name
aliases
category
quantity
unit
locationId
photoId
note
status
createdAt
updatedAt
```

`aliases` 用于支持“出国插头”和“旅行转换插头”等不同说法。

### InventoryEvent

```text
id
itemId
type
quantityDelta
fromLocationId
toLocationId
createdAt
note
```

操作类型包括：

```text
create
consume
add
move
adjust
delete
```

第一版可以不展示完整历史，但保留事件模型，方便未来实现撤销、库存历史和同步冲突处理。

## 5. 应用服务接口

页面只调用应用服务，不直接操作存储：

```text
createRoom()
updateRoom()
createLocation()
moveLocation()
deleteLocation()

createItem()
updateItem()
moveItem()
adjustItemQuantity()
deleteItem()

searchItems(keyword)
getLocationPath(locationId)
getLocationContents(locationId)

attachPhoto(ownerType, ownerId, file)
deletePhoto(photoId)
```

`getLocationPath` 统一生成如下位置路径：

```text
我的家 > 卧室 > 衣柜 > 左侧柜门 > 第二层 > 蓝色收纳盒
```

## 6. 页面规划

### 首页

- 房间列表
- 最近添加的物品
- 最近使用的位置
- 搜索入口

### 房间页

- 房间封面
- 位置树
- 房间照片
- 照片位置标记
- 新增位置按钮

### 位置详情页

- 当前路径
- 子位置
- 当前物品
- 添加物品
- 拍照
- 修改位置

### 物品详情页

- 物品名称和照片
- 数量与单位
- 所在位置
- 修改数量
- 移动位置
- 备注

### 搜索页

第一版支持物品名称、别名、房间名、位置名和备注的关键词搜索。

## 7. 照片和 3D 扩展

### 第一阶段：照片热点

在房间照片上添加矩形或圆形热点，每个热点绑定一个 `locationId`。点击热点后打开对应位置。

### 第二阶段：简易 2.5D

使用墙面、地面、柜体、抽屉和收纳盒等简单模型。每个模型对象绑定位置 ID：

```text
modelObjectId
locationId
transform
```

### 第三阶段：真实扫描模型

接入 iOS 或其他扫描能力后，增加房间模型、模型对象和变换信息。原有房间、位置和物品数据继续复用。

## 8. 搜索和对话

对话功能分为意图识别和数据查询两步：

```text
自然语言输入
  ↓
意图识别
  ↓
结构化查询
  ↓
本地数据查询
  ↓
生成回答
```

例如：

```json
{
  "action": "find_item",
  "keyword": "转换插头"
}
```

第一版使用本地关键词匹配。未来接入 AI 时，AI 只负责将自然语言转换为结构化查询，最终位置必须来自本地或云端真实数据。

## 9. 数据导出与迁移

第一版应支持导出全部本地数据：

```text
manifest.json
rooms.json
locations.json
items.json
photos/
models/
```

每次导出记录：

```text
schemaVersion
appVersion
exportedAt
```

建议同时提供导入、删除全部数据、查看本地占用和数据版本升级能力。未来接入云端时，可以使用导出的数据作为迁移来源。

## 10. 推荐开发阶段

### 阶段一：本地数据闭环

完成房间、位置树、物品、数量、本地搜索和本地图片。

### 阶段二：照片热点

完成房间照片、位置标记、标记点击和位置关联。

### 阶段三：体验优化

加入最近使用、批量添加、快速修改数量、移动物品、导入导出和错误处理。

### 阶段四：扩展能力

再考虑云端同步、家庭成员、AI 图片识别、自然语言查询、简易 3D 和 iOS 原生扫描。

## 11. 关键风险

- 单张照片无法可靠推断完整空间结构，因此第一版应允许手动修正。
- 图片原图和 3D 文件可能快速增加本地占用，需要缩略图和压缩策略。
- 物品移动后必须提供快速更新入口，否则数据会很快失真。
- AI 识别结果只能作为建议，物品名称和数量应允许用户确认。
- 本地数据丢失风险较高，导出和导入功能应尽早实现。
