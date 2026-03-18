# 基础系统速记卡（3 分钟版）

## 1. 一句话总述

> 这套基础系统本质上是 **“配置驱动 + 客户端多 Manager 分层组织 + 服务端权威状态校验/同步”** 的玩家资产与任务管理系统。

---

## 2. 先背这 5 句话

1. **`ItemManager` 管资产，`BagManager` 管布局**。
2. **`EquipManager` 不是独立仓库，而是装备槽位视图**。
3. **任务系统同时维护任务列表和 NPC 状态索引**。
4. **客户端 UI 不直接改底层状态，真正生效以后端为准**。
5. **这套系统最大的特点是共享引用和多层分工，而不是把逻辑全塞进 UI。**

---

## 3. 30 秒讲清整体架构

### 3.1 六层结构

- **定义层**：`ItemDefine / EquipDefine / QuestDefine`
- **模型层**：`Item / Quest / BagItem`
- **客户端管理层**：`ItemManager / BagManager / EquipManager / QuestManager`
- **客户端服务层**：`ItemService / QuesService`
- **UI 层**：`UIBag / UICharEquip / UIQuestSystem`
- **服务端权威层**：服务端 `ItemManager / EquipManager / QuestManager / QuestService`

### 3.2 一句话理解

> **配置定义规则，Manager 维护状态，Service 负责通信，UI 负责展示，服务端负责最终生效。**

---

## 4. 60 秒讲清三条主链路

### 4.1 道具 / 背包链路

```text
服务端加道具
  -> 服务端 ItemManager.AddItem
  -> StatusManager.AddItemChange
  -> 客户端 ItemManager.OnItemNotify
  -> BagManager.AddItem / Reset
  -> UIBag 刷新格子
```

压缩一句话：

> **道具数量变化先改资产，再同步到背包布局。**

### 4.2 装备链路

```text
UICharEquip 点击装备
  -> EquipManager.EquipItem
  -> ItemService.SendEquipItem
  -> 服务端 EquipManager 记录槽位 itemId
  -> ItemEquipResponse
  -> 客户端 EquipManager.OnEquipItem
  -> OnEquipChanged
  -> UICharEquip 刷新
```

压缩一句话：

> **装备本质上是给某个 Item 资产挂上一个槽位视图。**

### 4.3 任务链路

```text
玩家点 NPC
  -> QuestManager.OpenNpcQuest
  -> QuesService 发接取/提交请求
  -> 服务端 QuestService + QuestManager 处理
  -> 返回任务响应
  -> 客户端 QuestManager.RefreshQuestStatus
  -> UIQuestSystem / NPC 状态刷新
```

压缩一句话：

> **任务系统既要维护任务本身，也要维护 NPC 该显示什么状态。**

---

## 5. 最容易被追问的 4 个点

### 5.1 为什么 `ItemManager` 和 `BagManager` 要拆开

- `ItemManager` 表示玩家拥有什么
- `BagManager` 表示这些东西怎么摆在格子里
- 一个是资产层，一个是布局层

### 5.2 为什么装备系统不复制装备对象

- 当前是 `EquipManager` 直接引用 `ItemManager` 中的 `Item`
- 这样可以避免双份数据不同步
- 后续扩展耐久、强化时更容易维护一致性

### 5.3 为什么任务系统要维护 `npcQuests`

- 任务面板要的是“全部有效任务”
- NPC 交互要的是“当前这个 NPC 的状态”
- 所以需要额外一层按 NPC 分组的索引

### 5.4 客户端是不是权威状态源

不是。

- 客户端主要负责展示与交互
- 购买、穿装备、接任务、交任务最终都以后端结果为准

---

## 6. 当前实现边界只记这 4 点

1. **`BagManager.RemoveItem()` 还是空实现**，背包删减同步有风险。
2. **`Reset()` 没有明显先清空旧格子**，数量减少时可能残留旧格内容。
3. **装备服务端校验比较轻**，细粒度职业/槽位校验要确认是否在别处补了。
4. **任务状态刷新靠整体重建索引**，简单但不够增量化。

---

## 7. 最后 20 秒复盘模板

如果要快速收口，可以直接按这段说：

> 基础系统整体采用配置驱动和分层管理。`ItemManager` 管玩家资产，`BagManager` 管背包格子布局，`EquipManager` 管装备槽位视图，`QuestManager` 同时管理任务列表和 NPC 状态索引。客户端通过 `ItemService`、`QuesService` 发请求，真正的购买、装备和任务状态都以后端处理结果为准。当前实现的重点边界在于背包删除逻辑、装备校验粒度以及任务状态刷新方式。

---

## 8. 对应完整版文档

- 架构总览：`Doc/基础系统/01_核心架构系统.md`
- 背包与道具：`Doc/基础系统/02_背包与道具系统.md`
- 装备系统：`Doc/基础系统/03_装备系统.md`
- 任务系统：`Doc/基础系统/04_任务系统.md`
- 核心类索引：`Doc/基础系统/05_核心代码类.md`
