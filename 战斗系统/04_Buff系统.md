# Buff系统分析

## 1. 概述

Buff系统采用 **服务端驱动** 架构，Buff的持续时间和伤害计算由服务端主导，客户端只负责显示。

## 2. Buff生命周期

```
Add (添加) -> Update (持续/跳伤害) -> Remove (移除)
   |              |                      |
   +-> 加属性     +-> DOT伤害            +-> 回滚属性
   +-> 加特效     +-> 计时               +-> 移除特效
```

## 3. 服务端Buff实现

### 3.1 Buff类

**Buff** (`Src/Server/GameServer/GameServer/Battle/Buff.cs`):
```csharp
class Buff
{
    public int BuffID;
    private Creature Owner;
    private BuffDefine Define;
    private BattleContext Context;
    public bool Stoped;
    private float time;
    private int hit;

    public Buff(int buffID, Creature owner, BuffDefine define, BattleContext context)
    {
        this.BuffID = buffID;
        this.Owner = owner;
        this.Define = define;
        this.Context = context;
        this.OnAdd();
    }
}
```

### 3.2 Buff添加

```csharp
private void OnAdd()
{
    // 添加特效
    if (this.Define.Effect != BuffEffect.None)
    {
        this.Owner.EffectMgr.AddEffect(this.Define.Effect);
    }
    
    // 添加属性
    AddAttr();

    // 广播Buff添加消息
    NBuffInfo buff = new NBuffInfo()
    {
        buffId = this.BuffID,
        buffType = this.Define.ID,
        casterId = this.Context.Caster.entityId,
        ownerId = this.Owner.entityId,
        Action = BuffAction.Add
    };
    Context.Battle.AddBuffAction(buff);
}
```

### 3.3 属性添加

```csharp
private void AddAttr()
{
    // 防御百分比加成
    if (this.Define.DEFRatio != 0)
    {
        this.Owner.Attributes.Buff.DEF += this.Owner.Attributes.Basic.DEF * this.Define.DEFRatio;
    }
    
    // 固定攻击加成
    if (this.Define.AD != 0)
    {
        this.Owner.Attributes.Buff.AD += this.Define.AD;
    }
    
    // 固定法强加成
    if (this.Define.AP != 0)
    {
        this.Owner.Attributes.Buff.AP += this.Define.AP;
    }
    
    // 重新计算最终属性
    this.Owner.Attributes.InitFinalAttributes();
}
```

### 3.4 Buff更新（DOT伤害）

```csharp
public void Update()
{
    if (Stoped) return;
    
    this.time += Time.deltaTime;
    
    // 间隔触发伤害
    if (this.Define.Interval > 0)
    {
        if (this.time > this.Define.Interval * (this.hit + 1))
        {
            this.DoBuffDamage();
        }
    }
    
    // 持续时间结束，移除Buff
    if (this.time > this.Define.Duration)
    {
        this.OnRemove();
    }
}
```

### 3.5 DOT伤害计算

```csharp
private void DoBuffDamage()
{
    this.hit++;
    NDamageInfo damage = this.CalcBuffDamage(Context.Caster);
    
    Log.InfoFormat("Buff[{0}].DoBuffDamage[{1}] Damage:{2} Crit:{3}", 
        this.Define.Name, this.Owner.Name, damage.Damage, damage.Crit);
    
    // 应用伤害
    this.Owner.DoDamage(damage, Context.Caster);
    
    // 广播Buff伤害消息
    NBuffInfo buff = new NBuffInfo()
    {
        buffId = this.BuffID,
        buffType = this.Define.ID,
        casterId = this.Context.Caster.entityId,
        ownerId = this.Owner.entityId,
        Action = BuffAction.Hit,
        Damage = damage
    };
    Context.Battle.AddBuffAction(buff);
}
```

### 3.6 Buff伤害公式

```csharp
private NDamageInfo CalcBuffDamage(Creature caster)
{
    // 基础伤害 + 施法者属性加成
    float ad = this.Define.AD + caster.Attributes.AD * this.Define.ADFactor;
    float ap = this.Define.AP + caster.Attributes.AP * this.Define.APFactor;

    // 防御减免
    float addmg = ad * (1 - Owner.Attributes.DEF / (Owner.Attributes.DEF + 100));
    float apdmg = ap * (1 - Owner.Attributes.MDEF / (Owner.Attributes.MDEF + 100));

    float final = addmg + apdmg;
    
    NDamageInfo damageInfo = new NDamageInfo();
    damageInfo.Damage = Math.Max(1, (int)final);
    damageInfo.entityId = this.Owner.entityId;
    return damageInfo;
}
```

### 3.7 Buff移除

```csharp
private void OnRemove()
{
    // 回滚属性
    RemoveAttr();
    
    Stoped = true;
    
    // 移除特效
    if (this.Define.Effect != BuffEffect.None)
    {
        this.Owner.EffectMgr.RemoveEffect(this.Define.Effect);
    }
    
    // 广播Buff移除消息
    NBuffInfo buff = new NBuffInfo()
    {
        buffId = this.BuffID,
        buffType = this.Define.ID,
        casterId = this.Context.Caster.entityId,
        ownerId = this.Owner.entityId,
        Action = BuffAction.Remove
    };
    Context.Battle.AddBuffAction(buff);
}
```

### 3.8 属性回滚

```csharp
private void RemoveAttr()
{
    if (this.Define.DEFRatio != 0)
    {
        this.Owner.Attributes.Buff.DEF -= this.Owner.Attributes.Basic.DEF * this.Define.DEFRatio;
    }
    if (this.Define.AD != 0)
    {
        this.Owner.Attributes.Buff.AD -= this.Define.AD;
    }
    if (this.Define.AP != 0)
    {
        this.Owner.Attributes.Buff.AP -= this.Define.AP;
    }
    
    // 重新计算最终属性
    this.Owner.Attributes.InitFinalAttributes();
}
```

## 4. Buff管理器

**BuffManager** (`Src/Server/GameServer/GameServer/Battle/BuffManager.cs`):
```csharp
class BuffManager
{
    private Creature Owner;
    List<Buff> Buffs = new List<Buff>();
    private int idx = 1;
    private int BuffID { get { return this.idx++; } }

    public BuffManager(Creature owner)
    {
        this.Owner = owner;
    }

    internal void AddBuff(BattleContext context, BuffDefine buffDefine)
    {
        Buff buff = new Buff(this.BuffID, this.Owner, buffDefine, context);
        Buffs.Add(buff);
    }

    public void Upate()
    {
        for (int i = 0; i < Buffs.Count; i++)
        {
            if (!this.Buffs[i].Stoped)
            {
                this.Buffs[i].Update();
            }
        }
        // 清理已停止的Buff
        this.Buffs.RemoveAll((b) => b.Stoped);
    }
}
```

## 5. 客户端Buff实现

### 5.1 客户端Buff类

**Buff** (`Src/Client/Assets/Scripts/Battle/Buff.cs`):
```csharp
public class Buff
{
    public bool Stoped = false;
    private Creature Owner;
    public int BuffId;
    public BuffDefine Define;
    private int CasterId;
    public float time;

    public Buff(Creature owner, int buffId, BuffDefine define, int casterId)
    {
        Stoped = false;
        this.Owner = owner;
        this.BuffId = buffId;
        this.Define = define;
        this.CasterId = casterId;
        this.OnAdd();
    }
}
```

### 5.2 客户端Buff添加

```csharp
private void OnAdd()
{
    Debug.LogFormat("Buff[{0}:{1}]OnAdd", this.BuffId, this.Define.Name);
    
    // 添加特效
    if (this.Define.Effect != BuffEffect.None)
    {
        this.Owner.AddBuffEffect(this.Define.Effect);
    }
    
    // 添加属性（本地预测）
    AddAttr();
}
```

### 5.3 客户端Buff更新

```csharp
public void OnUpdate(float delta)
{
    if (Stoped) return;
    
    this.time += delta;
    
    // 持续时间结束
    if (this.time > this.Define.Duration)
    {
        this.OnRemove();
    }
}
```

### 5.4 客户端Buff移除

```csharp
public void OnRemove()
{
    Debug.LogFormat("Buff[{0}:{1}]OnRemove", this.BuffId, this.Define.Name);
    
    // 回滚属性
    RemoveAttr();
    
    Stoped = true;
    
    // 移除特效
    if (this.Define.Effect != BuffEffect.None)
    {
        this.Owner.RemoveBuffEffect(this.Define.Effect);
    }
}
```

## 6. Buff触发时机

Buff可以通过不同的触发时机附加到目标：

**TriggerType 枚举**:
- `SkillCast`: 技能施放时触发
- `SkillHit`: 技能命中时触发

**技能中触发Buff** (`Src/Server/GameServer/GameServer/Battle/Skill.cs`):
```csharp
private void AddBuff(TriggerType trigger, Creature target)
{
    if (this.Define.Buff == null || this.Define.Buff.Count == 0)
        return;

    foreach (var buffId in this.Define.Buff)
    {
        var buffDefine = DataManager.Instance.Buffs[buffId];
        
        // 检查触发类型
        if (buffDefine.Trigger != trigger)
            continue;
        
        // 根据目标类型添加Buff
        if (buffDefine.Target == TargetType.Self)
        {
            this.Owner.AddBuff(this.Context, buffDefine);
        }
        else if (buffDefine.Target == TargetType.Target)
        {
            target.AddBuff(this.Context, buffDefine);
        }
    }
}
```

## 7. BuffAction 类型

**BuffAction 枚举**:
- `Onne`: 协议默认值（命名历史拼写，语义上可视为 None）
- `Add`: Buff添加
- `Remove`: Buff移除
- `Hit`: Buff触发伤害（DOT）

## 8. 关键设计要点

1. **服务端权威**：Buff的持续时间和伤害计算完全由服务端控制
2. **属性回滚机制**：Buff移除时通过减法回滚属性，确保数据一致性
3. **Stoped标记**：使用Stoped标记而非立即删除，避免遍历时修改集合
4. **DOT间隔触发**：通过 hit 计数器控制DOT触发次数
5. **防御减免**：DOT伤害同样遵循防御减免公式
6. **客户端预测**：客户端也维护Buff状态用于显示，但最终以服务端为准

### 8.1 维护注意点

- 客户端本地也会按持续时间触发移除，用于 UI/表现连续性；最终状态仍以服务端 `BuffResponse` 为准。
- 若出现短时状态闪烁，优先检查服务端 Buff 广播延迟与客户端本地计时是否冲突。
