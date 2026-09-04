# 杀戮尖塔2 事件系统 — 配置与执行复刻说明

> 目标读者：想要在自己的实现（无头模拟器 / 复刻引擎）里**一比一复刻 STS2 事件系统**的人。
> 本文同时对照两套源码：
> - **原版**：`decompiled/MegaCrit.Sts2.Core.*`（Godot + C# 反编译代码）
> - **本项目复刻**：`sts2_env/`（Python 无头模拟器，已覆盖全部 68 个事件）
>
> 阅读建议：先看 §1 全景图，再按 §2(配置) → §3(执行) 读原版，最后用 §4 对照本项目实现、§6 的 Cookbook 落地代码。

---

## 1. 概念速览

STS2 的事件系统在**运行时**上其实是一个非常轻量的模型：

> 地图上的一个节点被判定为 `Event` 房间 → 进入 `EventRoom` → 交给一个 **事件实例（`EventModel`）** → 事件根据当前 Run 状态**现场生成一组选项（`EventOption`）** → 玩家点选 → 触发该选项绑定的**回调闭包**执行效果 → 回调里决定：**进入下一页 / 直接结束 / 插入一场战斗后再回来**。

事件本身没有独立的"配置文件"（不像 STS1 的 `*.json` 事件表）。它的"配置"**即代码本身**：每个事件是一个继承 `EventModel` 的 C# 类，反编译目录 `MegaCrit.Sts2.Core.Models.Events/` 下有 **约 70 个**这样的事件类；其中一小撮是特殊子类 `AncientEventModel`（古神/祭坛事件：Neow、Darv、Pael、Tanx、Vakuu、Orobas、Tezcatara、Nonupeipe…），其状态由 "Ancient" 槽位决定（见 §2.5）。

术语速查（原版 → 本项目）：

| 原版 (C#) | 本项目 (Python) | 作用 |
|---|---|---|
| `EventModel`（每事件一个子类） | `EventModel` 子类（`sts2_env/events/*.py`） | 事件的"配置+执行"载体 |
| `EventOption` | `run/events.py` 里的 `EventOption` dataclass | 一页里的一个可选按钮 |
| `EventModel.GenerateInitialOptions()` | `generate_initial_options(run_state)` | 进入事件时生成第一页选项 |
| 选项回调 `Func<Task> OnChosen` | `choose(run_state, option_id)` 分发 | 点选后的效果执行 |
| `SetEventState / SetEventFinished` | `EventResult(finished, next_options)` | 翻页 / 结束的返回值 |
| `ModelDb.Event<T>()` + canonical 单例 | `register_event(...)` 注册表 | 事件目录 |
| `ActModel` 内 `RoomSet.events`（每幕洗牌事件池） | `map/acts.py` 的 `ActConfig.event_ids` | 该幕能出哪些事件、顺序 |
| `RoomSet.EnsureNextEventIsValid` | `run/events.py: pick_event()` | 抽下一个"可用且未访问"的事件 |
| `RunManager` 进房/切房 | `run_manager.py` 各 `_enter_*` / `_do_*` | 房间级调度 |

两条主线贯穿全文：

1. **配置线（静态）**：一个事件"长什么样、什么时候出现、选项文案与数值从哪来、属于哪个池子"。
2. **执行线（动态）**：进入事件后"RNG 怎么种、选项怎么生成、点选后怎么改状态、怎么结束/回地图/进战斗"。

---

## 2. 静态配置层：事件是怎么"定义"出来的（原版）

### 2.1 事件 = 一个类文件

以 [TeaMaster.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Events/TeaMaster.cs)（15 行起）为例，一个最普通的事件只做三件事：

1. `protected override IEnumerable<EventOption> GenerateInitialOptions()` —— 决定第一页选项；
2. 为每个选项准备一个私有 async 回调（如 `BoneTea()`）；
3. `override bool IsAllowed(RunState)` —— 声明自己"什么时候允许出现在池子里"。

```csharp
public sealed class TeaMaster : EventModel
{
    protected override IEnumerable<EventOption> GenerateInitialOptions()
    {
        // 金币足够 → 给可点选项并绑定回调；否则给出"锁定"选项（OnChosen == null）
        if (Owner.Gold >= DynamicVars["BoneTeaCost"].BaseValue)
            list.Add(new EventOption(this, BoneTea, "TEA_MASTER.pages.INITIAL.options.BONE_TEA", ...));
        else
            list.Add(new EventOption(this, null, "TEA_MASTER.pages.INITIAL.options.BONE_TEA_LOCKED"));
        ...
    }

    private async Task BoneTea()   // 选项回调 = 效果本体
    {
        await PlayerCmd.LoseGold(DynamicVars["BoneTeaCost"].BaseValue, Owner, GoldLossType.Spent);
        await RelicCmd.Obtain<BoneTea>(Owner);
        SetEventFinished(L10NLookup("TEA_MASTER.pages.DONE.description")); // 结束本事件
    }
}
```

### 2.2 `EventModel` 的关键"配置位"（决定一个事件长什么样）

源码：[EventModel.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs)，基类成员语义如下：

| 成员 | 反编译位置 | 作用 / 配置约定 |
|---|---|---|
| `LocTable` = `"events"` | [L30/L54](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L30-L54) | 文案查表名，所有事件共用一个 `events` 本地化表 |
| `Title` / `InitialDescription` | [L56-L58](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L56-L58) | 本地化键约定：`<Id.Entry>.title`、`<Id.Entry>.pages.INITIAL.description` |
| `LayoutType` | [L131](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L131-L142) | `Default / Combat / Ancient / Custom`（枚举见 [EventLayoutType.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Events/EventLayoutType.cs)）。Combat = 事件自带战斗 UI；Custom = 自绘场景 |
| `IsShared` | [L62](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L62) | 共享事件（多人/多幕都出现），影响 RNG 播种与 `EventRoom` 预完成逻辑 |
| `IsAllowed(runState)` | [L261](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L261-L264) | 出场过滤条件（幕数、金币、牌组、遗物……），决定抽选时是否跳过 |
| `CanonicalVars` + `DynamicVars` | [L94-L108](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L94-L108) | 事件"洗数值"变量（如伤害/价格），进入事件后先 `CalculateVars` 再展示 |
| `GenerateInitialOptions()` | [L249](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L249)（抽象，子类必写） | 第一页选项的**现场生成**入口 |
| `CanonicalInstance / IsMutable` | [L144-L159](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L144-L159) | 模型"规范/可变"双态（见 §2.4） |
| 资源路径推导 | [L161-L169](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L161-L169) | 立绘/背景/VFX 场景均按 `Id.Entry` 小写自动找 `events/…` 资源，纯美术层 |
| `StateChanged` 事件 | [L171-L173](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L171-L173) | 每次 `SetEventState` 后触发，UI 订阅它刷新页面 |

> 结论：**文案走 `events` 本地化表，美术资源按命名规则自动找，逻辑全部在 C# 类里**。因此纯逻辑复刻只需要关心类结构与 `GenerateInitialOptions` 的执行结果，不需要复刻资源。

### 2.3 `EventOption`：一个选项的配置

源码：[EventOption.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Events/EventOption.cs)：

| 字段/方法 | 位置 | 语义 |
|---|---|---|
| `TextKey` + 标题/描述 | [L14-L18、L56-L78](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Events/EventOption.cs#L14-L78) | 文案键指向 `events.<Entry>.pages.<PAGE>.options.<NAME>.title/description` |
| `OnChosen`（`Func<Task>`） | [L20](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Events/EventOption.cs#L20) | **`OnChosen == null` 即 `IsLocked`（锁定/不可选按钮）**，如 `BONE_TEA_LOCKED` |
| `IsLocked / DisableOnChosen / WasChosen` | [L24-L32](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Events/EventOption.cs#L24-L32) | 锁定、点选后禁用、只执行一次 |
| `Chosen()` | [L104-L115](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Events/EventOption.cs#L104-L115) | UI 点选入口：先跑 `BeforeChosen` 再跑 `OnChosen` |
| `RelicOption(...)` 工厂（在 EventModel 上） | [EventModel.cs L493-L505](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L493-L505) | 最常见选项：送/换一个遗物；标题缺失时回退到遗物自身文案 |
| `ThatDoesDamage / ThatWillKillPlayerIf` | [L123-L132](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Events/EventOption.cs#L123-L132) | 给选项打上"会造成致命伤害"标记（UI 红色警示） |
| `ThatWontSaveToChoiceHistory` | [L140](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Events/EventOption.cs#L140) | 不进 RunHistory（用于干扰选项） |
| `WithRelic` / `HoverTips` | [L80-L102](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Events/EventOption.cs#L80-L102) | 给按钮挂遗物展示/悬浮提示 |

### 2.4 注册与目录：`ModelDb` + `AbstractModel` 的"规范/可变"双态

- [AbstractModel.cs L28-L55](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/AbstractModel.cs#L28-L55)：所有内容模型（卡/遗物/事件/遭遇…）的公共基类。每种类型全局只允许**一个"规范(canonical)"实例**（构造函数里做查重 `DuplicateModelException`），`ModelId` 由类型推导。
- 运行时要修改状态时调用 `ToMutable()` → [MutableClone L102-L109](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/AbstractModel.cs#L102-L109)（浅拷贝 + 深拷贝引用字段），保证每次进事件都不污染共享单例。
- 访问入口：`ModelDb.Event<T>()` / `ModelDb.AncientEvent<T>()`，见 [ModelDb.cs L389-L397](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/ModelDb.cs#L389-L397)；类型枚举靠 `[GenerateSubtypes]` 源生成 + Mod 反射（[L52-L60](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/ModelDb.cs#L52-L60)）。
- 两个全集（本项目的 Python 事件池就来源于它们）：
  - [ModelDb.AllSharedEvents](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/ModelDb.cs#L97-L117)：18 个跨幕共享事件（BrainLeech、CrystalSphere、Wellspring 之外的共享池…）；
  - `ActModel.AllEvents`（[ActModel.cs L139](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/ActModel.cs#L139)）：每幕自己的事件全集。

### 2.5 事件池与"出场配置"：每幕洗一次牌，环形队列抽取

**关键文件：[ActModel.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/ActModel.cs)、[RoomSet.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Rooms/RoomSet.cs)、RunManager。**

事件不是"走到 ? 节点时从全量事件里加权随机"，而是**在 Run 开局就把每幕的事件排好队**：

1. Run 开始 → `RunManager.GenerateRooms()`（[L497-L521](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Runs/RunManager.cs#L497-L521)），对每幕调用 `ActModel.GenerateRooms(rng, unlockState, …)`。
2. `GenerateRooms()`（[ActModel.cs L224-L279](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/ActModel.cs#L224-L279)）：
   - 事件池 = `AllEvents(本幕) + ModelDb.AllSharedEvents(共享)`；
   - 未解锁 epoch 的事件先被剔除（L228-L239）；
   - 剩下的 `UnstableShuffle(rng)` 后塞进 `RoomSet.events`（每幕**洗牌一次**，事件之间等权重、无重复）；
   - 同一函数里还决定了 Boss、普通怪队列、精英队列，以及**该幕的 Ancient（古神）槽位** `_rooms.Ancient`（L278，从"本幕古神 + 解锁的共享古神"里抽一个）。
3. 后续每次需要事件时只做两件事（[ActModel.cs L327-L333](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/ActModel.cs#L327-L333)）：
   - `PullNextEvent()` → 先 `RoomSet.EnsureNextEventIsValid()`（[RoomSet.cs L104-L119](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Rooms/RoomSet.cs#L104-L119)）：从 `events[eventsVisited]` 开始循环，**跳过 `!IsAllowed` 或已被访问过（`VisitedEventIds`）的事件**，找不到就 Warn"允许重复"兜底；
   - 记录 `runState.AddVisitedEvent(...)`（**同一个事件一局只出现一次**）。
4. 古神事件不走上面队列：地图 `Ancient` 点直接 `PullAncient()` 返回 `_rooms.Ancient`（[ActModel.cs L322-L325](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/ActModel.cs#L322-L325)）。

> 注意"环形队列"细节：`RoomSet.NextEvent => events[eventsVisited % events.Count]`（[RoomSet.cs L64](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Rooms/RoomSet.cs#L64)），`MarkVisited(RoomType.Event)` 递增 `eventsVisited`（[L82-L102](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Rooms/RoomSet.cs#L82-L102)）。配合"全 Run 级 visited"过滤，实现了"每幕顺序洗牌 + 全局不重复"，这是**决定论复刻里最容易做错的一环**。

### 2.6 地图节点怎么变成"事件房间"

`RunManager.EnterMapPointInternal`（[L606-L666](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Runs/RunManager.cs#L606-L666)）→ `RollRoomTypeFor(pointType)`（[L695-L714](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Runs/RunManager.cs#L695-L714)）：

- 节点类型是 **Unknown(?)**：交给 Odds 系统 `State.Odds.UnknownMapPoint.Roll(blacklist, State)` 掷出房间类型；
- 节点类型是 **Ancient**：固定 `RoomType.Event`；
- `CreateRoom(RoomType.Event, mapPointType)`（[L668-L693](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Runs/RunManager.cs#L668-L693)）：
  - `mapPointType == Ancient` → `State.Act.PullAncient()`
  - 否则 → `State.Act.PullNextEvent(State)` → 得到 canonical `EventModel` 后构造 `new EventRoom(event)`。

### 2.7 文案/本地化约定（复刻时只需知道键结构）

- 事件标题键：`<ENTRY>.title`
- 首页正文：`<ENTRY>.pages.INITIAL.description`
- 选项文案：`<ENTRY>.pages.INITIAL.options.<NAME>.title / .description`
- 多页事件：把 `INITIAL` 换成其它页码（如 `DONE`、`EVENT`、`CURSED`…），用 `SetEventState` 切换，见 [Neow.cs L263-L274](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Events/Neow.cs#L263-L274) 与 [EventModel.L10NLookup](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L357-L360)。
- 自动键：`OptionKey` 用类型名 slug 生成（[EventModel.cs L507-L515](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L507-L515)）；很多事件直接手写完整键（TeaMaster 里就是 `TEA_MASTER.pages.…`）。

---

## 3. 运行时执行链路（原版）

### 3.1 进入事件房间的完整时序

```mermaid
sequenceDiagram
    participant M as RunManager
    participant R as EventRoom
    participant S as EventSynchronizer
    participant E as EventModel (mutable)
    participant U as UI / NEventRoom

    M->>M: RollRoomTypeFor → RoomType.Event
    M->>M: Act.PullNextEvent/Ancient（跳过不可用/已访问）
    M->>R: EnterRoom(new EventRoom(canonicalEvent))
    R->>R: Preload 事件资源（图/Layout 场景）
    R->>S: EventSynchronizer.BeginEvent(canonical, isPreFinished, onStart)
    S->>E: canonical.ToMutable() → mutable 实例
    E->>E: Owner=player; Rng = f(runSeed, netId, entryHash)
    E->>E: BeforeEventStarted()
    E->>E: CalculateVars()（洗伤害/价格等动态变量）
    E->>E: GenerateInitialOptionsWrapper → GenerateInitialOptions()
    E->>E: SetEventState(InitialDescription, options) → StateChanged
    E-->>U: UI 展示标题/正文/选项
    U->>E: 点某个 EventOption.Chosen()
    E->>E: BeforeChosen → OnChosen() 执行效果
    alt 回调里 SetEventState(下一页)
        E-->>U: 刷新页面（多页流程）
    else 回调里 SetEventFinished(结束文案)
        E->>E: IsFinished=true; EnsureCleanup→OnEventFinished()
        M-->>M: 离开房间，回地图 / 进下一个流程
    else 回调里 EnterCombatWithoutExitingEvent(...)
        E->>M: 开 CombatRoom（记住 ParentEventId + shouldResume）
        M-->>R: 战斗结束 Resume(exitedRoom) → 事件"续命"回到原页
    end
```

关键代码位置：

| 步骤 | 位置 |
|---|---|
| 进入房间 `EventRoom.Enter` | [EventRoom.cs L47-L74](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Rooms/EventRoom.cs#L47-L74) |
| 事件实例启动 `EventModel.BeginEvent` | [EventModel.cs L185-L213](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L185-L213) |
| 首屏初始化（含 null 选项兜底） | [SetInitialEventState / GenerateInitialOptionsWrapper / ReplaceNullOptions L215-L247](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L215-L247) |
| 切页 / 结束 | [SetEventState L437-L456](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L437-L456) / [SetEventFinished L407-L412](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L407-L412) |
| 事件内嵌战斗 | [EnterCombatWithoutExitingEvent L458-L491](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L458-L491) |
| 战斗结束回到事件 | [RunManager.cs L649-L654](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Runs/RunManager.cs#L649-L654) 恢复 `EventRoom(ParentEventId)`；`EventRoom.Resume` L90-L96 |
| 离开事件（清理） | [EventRoom.Exit L76-L88](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Rooms/EventRoom.cs#L76-L88) → `EnsureCleanup()`（保证 `OnEventFinished` 只调一次） |

### 3.2 事件自身 = 一个"页面状态机"

复刻时记住事件只暴露三种内部动作，全部由**选项回调内部**调用：

1. `SetEventState(desc, newOptions)` —— 换下一页（旧选项清空、可选 0 项时自动 `IsFinished=true`）；
2. `SetEventFinished(desc)` —— 结束（选项清空 + `EnsureCleanup`）；
3. `EnterCombatWithoutExitingEvent(encounter, extraRewards, shouldResume)` —— 插入一场战斗，打完可回到事件（多页示例见 [Neow.cs L263-L274](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Events/Neow.cs#L263-L274)，战斗/古神类事件则覆写 `CanonicalEncounter` + `LayoutType = Combat`）。

### 3.3 决定论与 RNG

每个事件实例在 `BeginEvent` 时**用当前 run 的 RNG 播种一把独立 `Rng`**（[EventModel.cs L193](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L193)）：

```
seed = runState.Rng.Seed + (IsShared ? 0 : 玩家NetId) + hash(Id.Entry)
```

因此**同一个种子下，事件的随机内容（选项池、数值）完全可复现**；单机复刻里 NetId 可当作玩家编号固定值。离开事件时，非共享事件还会做 checksum 校验（[EventRoom.Exit L78-L81](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Rooms/EventRoom.cs#L78-L81)）。

---

## 4. 本项目 Python 复刻对照

### 4.1 文件速查

| 职责 | 本项目 Python | 对应原版 C# |
|---|---|---|
| 事件引擎（基类/选项/结果/注册/抽选） | [sts2_env/run/events.py](file:///c:/Project/sts2-rl-agent/sts2_env/run/events.py) | `Events/EventOption.cs`、`Models/EventModel.cs`、`Rooms/RoomSet.cs` |
| 事件内容（~68 个） | [events/shared.py](file:///c:/Project/sts2-rl-agent/sts2_env/events/shared.py) [act1.py](file:///c:/Project/sts2-rl-agent/sts2_env/events/act1.py) [act2.py](file:///c:/Project/sts2-rl-agent/sts2_env/events/act2.py) [act3.py](file:///c:/Project/sts2-rl-agent/sts2_env/events/act3.py)（[__init__.py](file:///c:/Project/sts2-rl-agent/sts2_env/events/__init__.py) 负责 import 触发注册） | `Models.Events/*.cs` |
| 每幕事件池（静态清单） | [map/acts.py](file:///c:/Project/sts2-rl-agent/sts2_env/map/acts.py) `ACT_0/1/2.event_ids` | `ActModel.AllEvents + ModelDb.AllSharedEvents` |
| 事件抽选（洗牌+游标+visited） | [run/events.py `pick_event`](file:///c:/Project/sts2-rl-agent/sts2_env/run/events.py#L241-L274) | `PullNextEvent / EnsureNextEventIsValid` |
| 事件调度/执行 | [run/run_manager.py](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_manager.py) | `RunManager.cs`、`Rooms/EventRoom.cs` |
| 未知节点(? → 房间)概率 | [run/odds.py `UnknownMapPointOdds`](file:///c:/Project/sts2-rl-agent/sts2_env/run/odds.py#L36-L87) | `Odds/UnknownMapPointOdds.cs` |
| 每幕洗牌初始化 | [run/run_state.py `_generate_rooms`](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_state.py#L1512-L1518) | `ActModel.GenerateRooms` |
| 访问记录 / 去重 | [run/run_state.py `visited_event_ids`](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_state.py#L1430) | `RunState.VisitedEventIds` |

### 4.2 引擎基类的映射

[run/events.py](file:///c:/Project/sts2-rl-agent/sts2_env/run/events.py) 的 `EventModel`（L33 起）复刻了原版的行为，但把"选项回调闭包"改成了**集中式 `choose(option_id)`**：

| Python | 语义 | 对应原版 |
|---|---|---|
| `event_id` 类属性 | 事件唯一 ID（与 C# 类型名一致，如 `"TeaMaster"`） | `AbstractModel.Id.Entry` |
| `is_allowed(run_state)` | 出场过滤 | `EventModel.IsAllowed` |
| `generate_initial_options(run_state)` | 生成第一页 | `GenerateInitialOptions` |
| `choose(run_state, option_id) -> EventResult` | 执行选中项，返回结果 | 选项 `OnChosen` 回调 |
| `EventResult(finished, next_options, rewards, event_combat_setup, …)` | 结果 = 结束? / 下一页? / 奖励对象 / 内嵌战斗 | `SetEventFinished` / `SetEventState` / `EnterCombatWithoutExitingEvent` |
| `request_card_choice / request_multi_card_choice` | 让事件"挂起"等待选牌，`pending_choice` 记录 | 原版由独立选牌 UI/命令完成 |
| `register_event(...)` | 注册表（[L214-L228](file:///c:/Project/sts2-rl-agent/sts2_env/run/events.py#L214-L228)） | `ModelDb` canonical 实例缓存 |
| `get_rng / reset_rng_for_run / ensure_vars_calculated` | 事件专属 RNG + 每次运行重置 + 数值只算一次 | `BeginEvent` 播种 RNG + `DynamicVars` |
| `event_entry()` | 把 CamelCase ID 转成 `UPPER_SNAKE`（用于播种 hash，保持与 C# 键风格一致） | `StringHelper.Slugify(GetType().Name)` |

选项则简化成无闭包的纯数据（[L20-L30](file:///c:/Project/sts2-rl-agent/sts2_env/run/events.py#L20-L30)）：

```python
@dataclass
class EventOption:
    option_id: str      # 传给 choose() 的键，如 "bone_tea"
    label: str          # 按钮文案（本模拟器直接内嵌英文，无独立 loc 表）
    description: str = ""
    enabled: bool = True
```

`EventResult`（[L199-L209](file:///c:/Project/sts2-rl-agent/sts2_env/run/events.py#L199-L209)）驱动调度器分支：

```python
@dataclass
class EventResult:
    finished: bool = True                 # True → 事件结束
    next_options: list[EventOption] = []  # 非空 → 翻到下一页（finished=False 配合使用）
    rewards: dict = ...                   # reward_objects: 延迟到奖励屏发的奖励
    event_combat_setup: str | None = None # 非空 → 触发事件战斗
```

### 4.3 事件"三件套"示例对照（TeaMaster）

C# 原版见 [TeaMaster.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Events/TeaMaster.cs)，Python 复刻见 [events/act1.py L199-L259](file:///c:/Project/sts2-rl-agent/sts2_env/events/act1.py#L199-L259)：

| C# 原版 | Python 复刻 | 说明 |
|---|---|---|
| `IsAllowed`: `CurrentActIndex < 2 && 全员金币≥150` | `is_allowed`: `act_index < 2 and all(gold ≥ 150)`（L207-L211） | 完全一致 |
| `GenerateInitialOptions` 里金币不足 → `new EventOption(this, null, …LOCKED)` | `EventOption(..., enabled=gold>=cost)`（L216-L227） | Python 用 `enabled` 标记，调度层不展示禁用项 |
| 回调 `BoneTea()`: `PlayerCmd.LoseGold(50)` + `RelicCmd.Obtain<BoneTea>()` + `SetEventFinished` | `choose`: `lose_gold(...)` + `obtain_relic(...)` + `EventResult(finished=True)`（L232-L240） | 直接改 RunState |
| `DynamicVars`（价格可被 Mod 改） | 模块级常量 `_TEA_MASTER_BONE_TEA_COST = 50`（L31-L32） | 本项目固定常量（工程取舍） |
| loc 键 `TEA_MASTER.pages.…` | `label="Bone Tea (50g)"` 直接内嵌 | 无头模拟器不需要本地化 |

> BrainLeech 也是同样三板斧（[act1.py L37-L93](file:///c:/Project/sts2-rl-agent/sts2_env/events/act1.py#L37-L93)），且是"奖励延迟发放"的典型：`rewards={"reward_objects":[CardReward(...)]}` 把"选 1 张角色牌 / 拿无色牌奖励"交给通用奖励屏处理（对应原版 CardReward）。

### 4.4 事件执行编排（RunManager）

Python 用 `RunManager` 的大状态机驱动一切（阶段常量 [PHASE_EVENT L223](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_manager.py#L223)）：

1. 地图移动 `_do_map_move`（[L1194-L1228](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_manager.py#L1194-L1228)）：解析坐标 → `resolve_room_type(point_type)`（[run_state.py L1692-L1709](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_state.py#L1692-L1709)，`ANCIENT → EVENT`、`UNKNOWN → odds.roll`）→ `_enter_room(room_type)`。
2. `_enter_room`（[L813-L847](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_manager.py#L813-L847)）路由到 `_enter_event`（[L775-L790](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_manager.py#L775-L790)）：
   ```python
   event = pick_event(self._run_state, pool=act_cfg.event_ids)
   event.reset_rng_for_run(run_state); event.before_event_started(run_state)
   event.ensure_vars_calculated(run_state)
   self._event_options = event.generate_initial_options(run_state)
   ```
3. 动作出口 `_actions_event`（[L1151-L1182](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_manager.py#L1151-L1182)）：若有 `pending_choice` 则暴露选牌动作，否则逐个暴露 `event_choice`。
4. 执行入口 `_do_event_choice` / `_do_event_choose` / `_do_event_confirm_choice`（[L1831-L1859](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_manager.py#L1831-L1859)）→ `event.choose(...)`。
5. **`_apply_event_result`（[L1861+](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_manager.py#L1861)）统一消费 EventResult**，是本项目与原版最不同的地方（把 C# 回调里的自由副作用，收敛成一张"决策表"）：

```
result.finished & 无 next_options        → on_event_finished + 离开(回地图/结算)
result 有 reward_objects                 → 转入 CARD_REWARD 奖励屏，结束后按需回到事件
result.finished=False & next_options     → 覆盖 _event_options 继续事件（多页）
result.event_combat_setup                → 转 _enter_event_combat（L568 起），
                                           post_combat_phase 决定战斗后回事件还是结束
event.pending_choice 存在                → 先挂起，等玩家选完牌再 _resume_after_run_choice
```

开局 Neow：`run_manager` 初始化时若 `start_with_neow` 直接进 `_enter_neow`（[L292-L295](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_manager.py#L292-L295)、[L792-L802](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_manager.py#L792-L802)），从注册表取 `NEOW_EVENT_ID = "Neow"`（[L86](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_manager.py#L86)），其实现位于 [events/act3.py Neow L250 注册](file:///c:/Project/sts2-rl-agent/sts2_env/events/act3.py#L250)。

### 4.5 事件池：静态清单 + 每局一次洗牌

- **静态清单**：[map/acts.py](file:///c:/Project/sts2-rl-agent/sts2_env/map/acts.py) 里 `ACT_0 / ACT_1 / ACT_2` 的 `event_ids`（L54-L69 / L84-L106 / L121-L141），约 42/60/56 个 ID，等效于原版"本幕事件 + 共享事件"并集（本项目直接手写展开）。
- **抽选**：`pick_event`（[run/events.py L241-L274](file:///c:/Project/sts2-rl-agent/sts2_env/run/events.py#L241-L274)）循环跳过 `visited_event_ids` 与 `is_allowed=False` 的事件，等价于 `EnsureNextEventIsValid` + `AddVisitedEvent`。
- **洗牌时机**：`RunState._generate_rooms`（[run_state.py L1512-L1518](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_state.py#L1512-L1518)）开局对每幕 `self.rng.up_front.shuffle(act.event_ids)` 并重置游标 `events_visited=0`。

### 4.6 与原版的主要差异（工程取舍清单）

| # | 差异 | 原版 | 本项目 |
|---|---|---|---|
| 1 | 古神/Ancient 处理 | 每幕一个独立 Ancient 槽位 `_rooms.Ancient`，地图 Ancient 节点直接取 | 无独立槽位：把 Darv/Pael/Vakuu/Tanx/Orobas/Tezcatara/Nonupeipe 等**混入普通 `event_ids` 池**按序抽取 |
| 2 | 选项回调模型 | 每选项绑定闭包 `OnChosen` | 事件内 `choose(option_id)` 集中分发，选项是纯数据 |
| 3 | 文案 | 独立 `events` 本地化表 + DynamicVars | 直接内嵌英文字符串常量，数值用模块级常量 |
| 4 | 结束/翻页 | 回调内部直接 `SetEventState/SetEventFinished` | 回调返回 `EventResult`，由 `_apply_event_result` 统一驱动 |
| 5 | 选牌等子交互 | 独立 UI + ActionExecutor 命令 | `pending_choice` 挂起事件，`resolver` 回调收尾 |
| 6 | 奖励发放 | 效果即时用 `RelicCmd/PlayerCmd` 等命令执行；部分走 Reward | 双轨：可立即改状态，也可 `reward_objects` 走统一奖励屏（`_deferred_followup_rewards` 控制顺序） |
| 7 | 多人/同步 | `EventSynchronizer` + NetId 播种 RNG | 仅单人，`player_id` 顶替 NetId |
| 8 | RNG 种子 | `seed + NetId + hash(Entry)` | 同公式（`run/events.py create_event_rng` L78-L85） |

---

## 5. 端到端走读：一个"战斗后回事件"的例子

以 [BattlewornDummy](file:///c:/Project/sts2-rl-agent/sts2_env/events/shared.py#L657-L707)（打训练假人，3 档难度）为例串一遍 Python 侧的完整链路：

1. 走到某 `?`/事件节点 → `resolve_room_type` 掷出 `RoomType.EVENT` → `_enter_room(EVENT)`。
2. `_enter_event`：`pick_event` 命中 `BattlewornDummy`（ID 在三个幕的池子里都有）。
3. `generate_initial_options` 返回三档选项；UI/Agent 可见 `event_choice: setting_1/2/3`。
4. 选择 `setting_3` → `choose` 返回：
   ```python
   EventResult(finished=True, description="Started dummy training.",
               rewards={"reward_objects": [RelicReward(...)]},
               event_combat_setup="battleworn_dummy_v3")
   ```
5. `_apply_event_result` 见到 `event_combat_setup` → 清空事件页、`_enter_event_combat("battleworn_dummy_v3", …)`（run_manager L568 起）→ 打一场遭遇战。
6. 胜利后奖励屏发放 `RelicReward`；`post_combat_phase` 若为 None → 默认结束回地图（若事件要求"打完回到选项页"，对应 C# `EnterCombatWithoutExitingEvent` + `EventRoom.Resume`，Python 用 `post_combat_phase` / 续链回调模拟）。

> 另一个极端例子 `Wellspring`（[shared.py L2672-L2727](file:///c:/Project/sts2-rl-agent/sts2_env/events/shared.py#L2672-L2727)）展示**选牌挂起**：`Bathe` 选项先 `request_card_choice("Choose 1 card to remove", …)` 让玩家从牌组挑 1 张（对应 C# 里专门弹选牌界面），`resolver` 回调在选完后移除该牌并把 Guilty 诅咒塞进牌组（详见 [GUILTY_CARD_LOGIC.md](GUILTY_CARD_LOGIC.md)）。

---

## 6. 复刻 / 新增事件的 Cookbook

### 6.1 最小闭环需要实现的 4 个能力

无论用什么语言复刻，事件系统成立的前提是引擎已提供：

1. **房间调度**：能区分"这是个事件节点"，并能从地图流程进入/离开事件（不阻塞其它房间）。
2. **选项渲染/输入**：向玩家（或 Agent）展示当前页标题、正文、选项（含 disabled），拿到点选结果。
3. **副作用原语**：事件改状态需要的基础能力 —— 扣血/加血、金币、加/删/升级卡、发/换遗物、给药水、进遭遇战、结束游戏等（本项目在 `reward_objects.py` / `PlayerState` / `RelicCmd` 等价物上实现）。
4. **状态持久化**：visited 记录、事件内自定义变量（如多页事件走到第几页）在存档/读档后仍正确。

### 6.2 在本项目 Python 里新增一个事件（步骤）

以"抄 TeaMaster"为例：

1. 在合适的模块定义子类（`shared.py` / `act1.py`…，参考 [act1.py L199-L259](file:///c:/Project/sts2-rl-agent/sts2_env/events/act1.py#L199-L259)）：
   ```python
   class MyEvent(EventModel):
       event_id = "MyEvent"

       def is_allowed(self, run_state):          # 出场条件
           return run_state.current_act_index < 2

       def generate_initial_options(self, run_state):   # 第一页
           return [EventOption("option_a", "Option A", "desc"),
                   EventOption("option_b", "Option B", "desc", enabled=...)]

       def choose(self, run_state, option_id):   # 执行
           if option_id == "option_a":
               ... # 直接改状态 / 返回 rewards
               return EventResult(finished=True, description="...")
   ```
2. 在文件底部 `register_event(MyEvent())`（文件被 [events/__init__.py](file:///c:/Project/sts2-rl-agent/sts2_env/events/__init__.py) import，注册自动生效）。
3. 把 `"MyEvent"` 加进目标幕的 `event_ids` 清单（[map/acts.py](file:///c:/Project/sts2-rl-agent/sts2_env/map/acts.py)）。原版里等价操作是"继承 EventModel + 让 ActModel.AllEvents/AllSharedEvents 收录"。
4. 若事件用到随机，务必走 `self.get_rng(run_state)`（继承自带），不要直接用 Python 全局 random。
5. 涉及"选卡"的事件用 `request_card_choice(...)`；"打一架再回来"用 `event_combat_setup` + 可选的 `post_combat_phase`。

### 6.3 与原版一致性 Checklist（防"看起来对、跑起来不对"）

- [ ] `event_id` 与反编译 C# 类型名一致（如 `Wellspring`、`TeaMaster`），保证与数据/文档可对照。
- [ ] `is_allowed` 与原版 `IsAllowed` 条件逐字核对（幕数、金币、HP、牌组、遗物、访问记录都会影响）。
- [ ] 随机分支（出哪些遗物/选项、数值）**顺序**与原版一致 —— RNG 一次抽取的顺序错了，整局回放就偏。
- [ ] 事件结束后 `visited_event_ids` 必须加上（`pick_event` 已处理），否则同局会重复出现。
- [ ] 非结束返回要给 `next_options`（否则 `finished=False` 但无下一页会卡住）。
- [ ] 死亡分支：事件里玩家 HP 归零时，本项目的 `_apply_event_result` 会转 `RUN_OVER`（[L1879-L1882](file:///c:/Project/sts2-rl-agent/sts2_env/run/run_manager.py#L1879-L1882)）；原版在 `BeginEvent` 也有"进入即死"兜底（[EventModel.cs L198-L203](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L198-L203)）。
- [ ] 多页事件结束后要显式 `finished=True`，并在最后一页触发 `on_event_finished`（对应 C# `SetEventFinished → EnsureCleanup`）。

---

## 7. 附：最值得精读的 6 段原版代码

| 想搞清什么 | 看哪里 |
|---|---|
| 事件页状态机（Begin→首屏→切页→结束→清理） | [EventModel.cs L185-L456](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L185-L456) |
| 事件池排队/去重/兜底 | [RoomSet.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Rooms/RoomSet.cs) 全文 |
| 事件从地图/房间进来的调度 | [RunManager.cs L606-L714](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Runs/RunManager.cs#L606-L714) |
| 事件"打一场还能回来" | [EventModel.cs L458-L491](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/EventModel.cs#L458-L491) + [RunManager.cs L649-L654](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Runs/RunManager.cs#L649-L654) |
| 复杂多页 + 动态选项池 | [Neow.cs L192-L274](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Events/Neow.cs#L192-L274) |
| 每幕洗牌 + Ancient 槽位 | [ActModel.cs L224-L333](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/ActModel.cs#L224-L333) |
