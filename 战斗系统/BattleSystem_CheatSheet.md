# 战斗系统速记卡（3 分钟版）

## 1. 一句话总述

> 这套战斗系统本质上是 **“客户端预测表现 + 服务端权威结算 + 地图实例内 Tick 驱动推进”** 的战斗模拟系统。

---

## 2. 先背这 5 句话

1. **客户端先播，服务端结算**：客户端负责手感，服务端负责最终结果。
2. **战斗按地图实例隔离**：一个 `Map` 对应一个 `Battle`。
3. **请求先入队，再在 Tick 中执行**：不是收到请求就立刻结算。
4. **`Battle` 负责调度，`Creature / Monster / AI` 负责真正执行行为**。
5. **战斗结果按帧聚合，再广播给当前地图内玩家**。

---

## 3. 30 秒讲清架构

### 3.1 五层结构

- **接入层**：`BattleService`，接收 `SkillCastRequest`
- **路由层**：`BattleManager + MapManager`，定位到正确的 `Map / Battle`
- **调度层**：`GameServer + Map + Battle`，负责 Tick、队列和结果聚合
- **战斗对象层**：`Creature / Monster / AI / Skill / Buff`
- **输出层**：`Battle.BroadcastHitsMessage + Map.BroadcastBattleResponse`

### 3.2 核心关系

```text
MapManager
  -> Map
      -> Battle
```

关键词只记一句：

> **`Map` 提供地图上下文，`Battle` 提供战斗运行时。**

---

## 4. 60 秒讲清技能释放主链路

```text
客户端点技能
  -> BattleService.SendSkillCast
  -> 服务端 BattleService.OnSkillCast
  -> BattleManager 路由到 Map.Battle
  -> CastActions.Enqueue
  -> Battle.Update() 取出动作
  -> ExecuteAction
  -> Creature.CastSkill
  -> Skill.Cast / DoHit / Buff
  -> Battle 收集 Cast / Hit / Buff
  -> Map 广播给当前地图玩家
  -> 客户端收到 SkillHitResponse / BuffResponse 做表现
```

面试时可压缩成一句：

> **请求先入队，下一帧由 `Battle.Update` 消费，结算后统一广播。**

---

## 5. 属性、装备、Buff 怎么接到伤害公式里

### 5.1 属性模型

- **公式**：`Final = Basic + Buff`
- **展开**：`Basic = Initial + Growth + Equip`
- **战斗读取**：真正参与伤害计算的是 `Final`

### 5.2 装备作用

- 装备不会直接改 `Final`
- 装备先汇总到 `Equip`
- 再参与 `Basic` 和二级属性计算

### 5.3 Buff 作用

- Buff 改的是 `Attributes.Buff`
- 改完后调用 `InitFinalAttributes()`
- 所以 Buff 是“最终层叠加”，不是重跑整套初始化

---

## 6. Buff 系统只记这 3 点

1. **生命周期**：`Add -> Update -> Remove`
2. **服务端驱动**：持续时间、DOT 伤害、移除时机都以后端为准
3. **客户端职责**：主要负责图标、特效、扣血表现，不是权威状态源

---

## 7. 核心类只记最重要的

| 类 | 一句话定位 |
|---|---|
| `BattleService` | 战斗请求入口 |
| `BattleManager` | 把请求路由到正确战场 |
| `Map` | 地图实例上下文 |
| `Battle` | 战斗调度器，负责队列、Tick、广播 |
| `Creature` | 通用战斗单位 |
| `Monster` | 带移动和 AI 的战斗单位 |
| `Skill` | 技能校验、施放、命中、伤害计算 |
| `Attributes` | 属性分层计算核心 |
| `Buff` / `BuffManager` | Buff 生命周期与 DOT 推进 |

---

## 8. 面试最容易被追问的边界点

### 8.1 吞吐边界

- 当前 `Battle.Update()` 每 Tick 只处理 1 次施法动作
- 群战时要关注 `CastActions` 队列积压

### 8.2 多实例路由边界

- 当前路由主键主要是 `mapId`
- 多实例地图理论上还需要结合 `instanceId`

### 8.3 AOE 与持续更新边界

- AOE 查目标时可能查的是地图实体池
- 但目标不一定都进入 `AllUnits`
- “在地图里存在”不等于“进入战斗持续更新链路”

### 8.4 随机伤害边界

- 当前随机系数实现和常见“±5%浮动”写法不同
- 现实现会显著压低伤害，再由 `Math.Max(1, ...)` 托底

---

## 9. 最后 20 秒复盘模板

如果要在面试里快速收口，可以直接按这段说：

> 战斗系统整体是客户端预测、服务端权威结算。  
> 服务端按 `Map -> Battle` 做实例隔离，请求先入队，再由 `Battle.Update` 在 Tick 中消费。  
> `Battle` 负责调度，`Creature / Monster / AI / Skill / Buff` 负责真正执行。  
> 属性系统采用 `Initial / Growth / Equip / Buff / Final` 分层计算，伤害公式最终读的是 `Final`。  
> 当前实现要重点注意多实例路由、单 Tick 单动作吞吐和 AOE 单位池边界。

---

## 10. 对应完整版文档

- 架构总览：`Doc/战斗系统/01_核心架构系统.md`
- 技能时序：`Doc/战斗系统/02_技能释放流程系统.md`
- 属性装备：`Doc/战斗系统/03_属性与装备系统.md`
- Buff 机制：`Doc/战斗系统/04_Buff系统.md`
- 核心类索引：`Doc/战斗系统/05_核心代码类.md`
