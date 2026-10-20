# 杀戮尖塔2 怪物意图系统 — 配置说明与 C# 参考实现

> 目标读者：想要在自己的实现（无头模拟器 / 复刻引擎 / Mod）里**一比一复刻 STS2 怪物意图与 AI 出招**的人。
> 本文同时对照两套源码：
> - **原版**：`decompiled/MegaCrit.Sts2.Core.MonsterMoves.*`（Godot + C# 反编译代码）
> - **本项目复刻**：`sts2_env/monsters/`（Python 无头模拟器）
>
> 阅读建议：先看 §1 三层架构，再按 §2(意图) → §3(状态机) 读配置方式，最后用 §4 的 C# 参考实现落地。

---

## 1. 三层架构（核心结论）

STS2 的怪物 AI 分成三个**互相解耦**的层次：

| 层 | 原版类 | 本项目 Python | 职责 |
|---|---|---|---|
| 1. 意图 Intent | [AbstractIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/AbstractIntent.cs) 及子类 | [intents.py](file:///c:/Project/sts2-rl-agent/sts2_env/monsters/intents.py) | **纯展示数据**：图标、动画名、悬停说明、预期数值。不含任何游戏效果 |
| 2. 招式 MoveState | [MoveState.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.MonsterMoveStateMachine/MoveState.cs) | [state_machine.py `MoveState`](file:///c:/Project/sts2-rl-agent/sts2_env/monsters/state_machine.py#L39-L84) | 把「意图列表」和「实际执行逻辑（回调）」绑定在一起 |
| 3. 状态机 | [MonsterMoveStateMachine.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.MonsterMoveStateMachine/MonsterMoveStateMachine.cs) | [state_machine.py `MonsterAI`](file:///c:/Project/sts2-rl-agent/sts2_env/monsters/state_machine.py#L198-L264) | 用有向图描述「招式之间怎么流转」，即 AI 决策 |

### ⚠️ 最重要的坑：意图 ≠ 效果

意图和实际效果是**分离的**。`new SingleAttackIntent(15)` 只是告诉 UI 显示"攻击 15"，
**真正的伤害写在对应 move 方法里**的 `DamageCmd.Attack(15)`。游戏不会校验两者是否一致，
所以做复刻时**必须读 move 方法体，不能只信意图对象的数值**。

### 每回合的执行流程

对应 [MonsterModel.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/MonsterModel.cs#L313-L355)：

```
SetUpForCombat()          // 生成本场战斗的状态机实例
  → RollMove(targets)     // 沿状态图走到下一个 MoveState（决定"下回合的意图"）
  → PerformMove()         // 执行该 MoveState 的 onPerform 回调（真正落地效果）
```

`RollMove` 发生在**玩家回合开始时**（用于提前显示意图），`PerformMove` 发生在**敌方回合**。

---

## 2. 意图（Intent）：配置方式

### 2.1 意图类型清单（全部 15 项）

枚举定义见 [IntentType.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/IntentType.cs)。
按构造参数可分为三类：

| 意图类 | IntentType | 构造参数 | 说明 |
|---|---|---|---|
| [SingleAttackIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/SingleAttackIntent.cs#L8-L35) | Attack | `int damage` / `Func<decimal> damageCalc` | 单段攻击，显示伤害数字 |
| [MultiAttackIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/MultiAttackIntent.cs#L8-L42) | Attack | `(int damage, int repeat)` / `(int, Func<int>)` | 多段攻击，显示 `D×N` |
| [DeathBlowIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/DeathBlowIntent.cs#L10-L32) | DeathBlow | `Func<decimal> damageCalc` | 继承 SingleAttack，特殊图标 |
| [BuffIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/BuffIntent.cs) | Buff | 无 | 自身增益 |
| [DebuffIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/DebuffIntent.cs#L3-L27) | Debuff / DebuffStrong | `bool strong = false` | 弱化/强力版图标不同 |
| [DefendIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/DefendIntent.cs) | Defend | 无 | 格挡 |
| [StatusIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/StatusIntent.cs#L7-L36) | StatusCard | `int count` | 塞状态牌，显示张数 |
| [CardDebuffIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/CardDebuffIntent.cs) | CardDebuff | 无 | 对牌做减益 |
| [HealIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/HealIntent.cs) | Heal | 无 | 治疗 |
| [SummonIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/SummonIntent.cs) | Summon | 无 | 召唤 |
| [EscapeIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/EscapeIntent.cs) | Escape | 无 | 逃跑 |
| [SleepIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/SleepIntent.cs) | Sleep | 无 | 睡眠 |
| [StunIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/StunIntent.cs#L9-L27) | Stun | 无 | 眩晕 |
| [HiddenIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/HiddenIntent.cs#L3-L11) | Hidden | 无 | 无图标、无 tip（`HasIntentTip=false`） |
| [UnknownIntent.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/UnknownIntent.cs) | Unknown | 无 | 未知图标 |

### 2.2 一个招式可以挂多个意图

`MoveState` 的构造函数是 `params AbstractIntent[] intents`，所以可以并列多个，UI 横向排列：

```csharp
// 攻击 + 上弱化（SludgeSpinner 的 Oil Spray）
new MoveState("OIL_SPRAY_MOVE", OilSprayMove, new SingleAttackIntent(OilSprayDamage), new DebuffIntent());
```

### 2.3 攻击数值是**动态计算**的

`AttackIntent.GetSingleDamage()` 内部会走 `Hook.ModifyDamage(...)`（见
[AttackIntent.cs L84-L93](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.Intents/AttackIntent.cs#L84-L93)），
即**玩家看到的数字已经包含了力量/易伤等修正**。原版还支持传 `Func<decimal>`，
让伤害随战斗状态变化（例：Waterfall Giant 的必杀伤害 = 当前压力层数）。

---

## 3. 状态机：AI 决策图的配置方式

目录 [MonsterMoveStateMachine/](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.MonsterMoveStateMachine) 共 3 种节点，
都继承 [MonsterState.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.MonsterMoveStateMachine/MonsterState.cs)：

### 3.1 `MoveState` — 叶子节点（真正的招式）

关键成员（见 [MoveState.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.MonsterMoveStateMachine/MoveState.cs#L11-L53)）：

| 成员 | 作用 |
|---|---|
| `Intents` | 展示用意图列表 |
| `onPerform`（构造函数委托） | 实际执行逻辑 |
| `FollowUpState` / `FollowUpStateId` | 打完必去的下一状态；为空则回到 initial |
| `MustPerformOnceBeforeTransitioning` | 未执行过一次前无法离开（用于眩晕/蓄力回合，见 §5 案例 4） |

### 3.2 `RandomBranchState` — 加权随机分支

`AddBranch(state, cooldown, repeatType, weight)`，其中权重计算规则见
[RandomBranchState.cs L130-L167](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.MonsterMoveStateMachine/RandomBranchState.cs#L130-L167)：

`MoveRepeatType`（[定义](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves/MoveRepeatType.cs)）四种语义：

| 取值 | 含义 |
|---|---|
| `CanRepeatForever` | 无限制 |
| `CanRepeatXTimes` | 最多连续 N 次，之后权重归零直到被打断 |
| `CannotRepeat` | 不能连续两次（等价 `maxTimes = 1`） |
| `UseOnlyOnce` | 整场战斗只用一次（查 StateLog 去重） |

`cooldown > 0` 表示：最近 N 个 move 里若出现过本招则权重归零。

### 3.3 `ConditionalBranchState` — 条件分支

`AddState(move, () => 条件)`，按注册顺序返回**第一个成立**的分支（见
[ConditionalBranchState.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.MonsterMoveStateMachine/ConditionalBranchState.cs#L45-L60)）。

### 3.4 推进规则（易错点）

`MonsterMoveStateMachine.FindNextMoveState()` 的关键逻辑（见
[MonsterMoveStateMachine.cs L54-L80](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.MonsterMoves.MonsterMoveStateMachine/MonsterMoveStateMachine.cs#L54-L80)）：

1. 循环推进，**跳过所有分支节点**，直到落在 `MoveState` 上；
2. `_performedFirstMove` 为 false 且当前是 MoveState 时**不推进**（首回合锁招）；
3. 分支节点的 `ShouldAppearInLogs = false`，**不写入 StateLog**（StateLog 只记 move）；
4. `initialState` 允许直接是分支节点（如 Myte 的 `INIT_MOVE`）。

---

## 4. C# 参考实现（可独立取用）

下面是一份**去掉 Godot/UI 依赖**的精简 C# 模型，忠实复刻原版的意图与状态机骨架，可直接移植进自己的引擎。
所有关键行为都加了注释标注「对应原版哪里」。

```csharp
// ============================================================================
// STS2 怪物意图系统 —— 独立可用的 C# 参考实现
// 目标：只保留「意图配置 + 出招决策」核心，剔除 Godot 节点/动画/音效。
// ============================================================================

using System;
using System.Collections.Generic;
using System.Linq;

namespace Sts2.MonsterAi
{
    // ------------------------------------------------------------------------
    // 1) 意图类型枚举（对应原版 MonoCrit...IntentType，共 15 项）
    // ------------------------------------------------------------------------
    public enum IntentType
    {
        Attack,        // 单段攻击
        Buff,          // 自身增益
        Debuff,        // 普通减益
        DebuffStrong,  // 强力减益（换图标）
        Defend,        // 格挡
        Escape,        // 逃跑
        Heal,          // 治疗
        Hidden,        // 隐藏（无图标、无悬停提示）
        Summon,        // 召唤
        Sleep,         // 睡眠
        Stun,          // 眩晕
        StatusCard,    // 塞状态牌
        CardDebuff,    // 对牌减益
        DeathBlow,     // 必杀
        Unknown        // 未知
    }

    // ------------------------------------------------------------------------
    // 2) 意图基类 & 子类
    //    意图是「纯展示数据」：只描述会做什么，不含任何效果。
    // ------------------------------------------------------------------------
    public abstract class AbstractIntent
    {
        /// <summary>意图种类，用于观测编码（one-hot）与图标选择。</summary>
        public abstract IntentType IntentType { get; }
    }

    /// <summary>攻击意图基类。伤害用 Func&lt;int&gt; 以支持动态数值（如随层数增长）。</summary>
    public abstract class AttackIntent : AbstractIntent
    {
        protected readonly Func<int> DamageCalc;

        protected AttackIntent(Func<int> damageCalc) => DamageCalc = damageCalc;

        /// <summary>攻击段数（单段=1，多段=N）。</summary>
        public abstract int Repeats { get; }

        /// <summary>
        /// 单段伤害。原版这里会经过 Hook.ModifyDamage（力量/易伤修正），
        /// 若你的引擎已实现修正管线，应在此处调用；否则直接返回基础值。
        /// </summary>
        public virtual int SingleDamage => Math.Max(0, DamageCalc());

        /// <summary>总伤害 = 单段 × 段数。</summary>
        public int TotalDamage => SingleDamage * Repeats;
    }

    /// <summary>单段攻击。对应原版 SingleAttackIntent。</summary>
    public sealed class SingleAttackIntent : AttackIntent
    {
        public override IntentType IntentType => IntentType.Attack;
        public override int Repeats => 1;

        public SingleAttackIntent(int damage) : base(() => damage) { }
        public SingleAttackIntent(Func<int> damageCalc) : base(damageCalc) { }
    }

    /// <summary>多段攻击。对应原版 MultiAttackIntent。</summary>
    public sealed class MultiAttackIntent : AttackIntent
    {
        public override IntentType IntentType => IntentType.Attack;

        private readonly Func<int> _repeatCalc;
        public override int Repeats => _repeatCalc();

        public MultiAttackIntent(int damage, int repeat) : base(() => damage) => _repeatCalc = () => repeat;
        public MultiAttackIntent(int damage, Func<int> repeatCalc) : base(() => damage) => _repeatCalc = repeatCalc;
    }

    /// <summary>必杀：本质是单段攻击，但用不同图标。对应原版 DeathBlowIntent。</summary>
    public sealed class DeathBlowIntent : AttackIntent
    {
        public override IntentType IntentType => IntentType.DeathBlow;
        public override int Repeats => 1;

        public DeathBlowIntent(Func<int> damageCalc) : base(damageCalc) { }
    }

    /// <summary>无参数意图的通用实现：Buff/Defend/Heal/Summon/Escape/Sleep/Stun/Unknown 等。</summary>
    public sealed class SimpleIntent : AbstractIntent
    {
        public override IntentType IntentType { get; }
        public SimpleIntent(IntentType type) => IntentType = type;
    }

    /// <summary>减益意图。strong=true 时切换为 DebuffStrong。对应原版 DebuffIntent。</summary>
    public sealed class DebuffIntent : AbstractIntent
    {
        private readonly bool _strong;
        public override IntentType IntentType => _strong ? IntentType.DebuffStrong : IntentType.Debuff;
        public DebuffIntent(bool strong = false) => _strong = strong;
    }

    /// <summary>塞状态牌意图，携带张数。对应原版 StatusIntent。</summary>
    public sealed class StatusIntent : AbstractIntent
    {
        public override IntentType IntentType => IntentType.StatusCard;
        public int CardCount { get; }
        public StatusIntent(int count) => CardCount = count;
    }

    /// <summary>隐藏意图：不出图标、不显示提示。对应原版 HiddenIntent。</summary>
    public sealed class HiddenIntent : AbstractIntent
    {
        public override IntentType IntentType => IntentType.Hidden;
        public bool HasIntentTip => false;   // 原版同样覆写为 false
    }

    // ------------------------------------------------------------------------
    // 3) 状态机节点
    // ------------------------------------------------------------------------
    public abstract class MonsterState
    {
        public abstract string Id { get; }

        /// <summary>是否写入 StateLog。分支节点为 false（原版同名属性）。</summary>
        public virtual bool ShouldAppearInLogs => true;

        /// <summary>当前是否允许离开此状态。</summary>
        public virtual bool CanTransitionAway => true;

        /// <summary>是否是「招式」叶子节点。分支节点返回 false。</summary>
        public virtual bool IsMove => this is MoveState;

        /// <summary>返回下一状态的 Id。分支节点在这里做随机/条件选择。</summary>
        public abstract string GetNextState(IReadOnlyList<string> stateLog);

        public virtual void OnEnterState() { }
        public virtual void OnExitState() { }
    }

    /// <summary>
    /// 招式节点：绑定意图列表 + 执行回调。
    /// 对应原版 MoveState（见 decompiled/.../MoveState.cs）。
    /// </summary>
    public sealed class MoveState : MonsterState
    {
        private bool _performedAtLeastOnce;

        public override string Id { get; }
        public IReadOnlyList<AbstractIntent> Intents { get; }

        /// <summary>实际执行逻辑。参数是该招式的目标（玩家方单位）。</summary>
        private readonly Action _onPerform;

        /// <summary>打完必去的下一状态；为 null 时由状态机回落到 initial。</summary>
        public string? FollowUpStateId { get; init; }

        /// <summary>未执行过一次前不能离开（眩晕/蓄力回合用）。</summary>
        public bool MustPerformOnceBeforeTransitioning { get; set; }

        public MoveState(string stateId, Action onPerform, params AbstractIntent[] intents)
        {
            Id = stateId;
            _onPerform = onPerform;
            Intents = intents;
        }

        // 覆盖默认的 CanTransitionAway：未执行过就不能走（原版逻辑）
        public override bool CanTransitionAway =>
            !MustPerformOnceBeforeTransitioning || _performedAtLeastOnce;

        /// <summary>执行本招式的真实效果。</summary>
        public void Perform()
        {
            _performedAtLeastOnce = true;
            _onPerform();
        }

        // 离开时重置「已执行过」标记（原版 OnExitState）
        public override void OnExitState() => _performedAtLeastOnce = false;

        public override string GetNextState(IReadOnlyList<string> stateLog) =>
            FollowUpStateId ?? throw new InvalidOperationException($"'{Id}' 未配置 FollowUpState");
    }

    /// <summary>重复约束类型。对应原版 MoveRepeatType。</summary>
    public enum MoveRepeatType
    {
        CanRepeatForever,  // 无限制
        CanRepeatXTimes,   // 最多连续 N 次
        CannotRepeat,      // 不能连续两次（等价 maxTimes=1）
        UseOnlyOnce        // 整场只用一次
    }

    /// <summary>随机分支的一条候选。对应原版 RandomBranchState.StateWeight。</summary>
    public sealed class WeightedBranch
    {
        public string StateId { get; init; } = "";
        public MoveRepeatType RepeatType { get; init; } = MoveRepeatType.CanRepeatForever;
        public int MaxTimes { get; init; } = 1;
        public int Cooldown { get; init; } = 0;
        private readonly Func<float> _weightCalc;
        public float BaseWeight => _weightCalc();

        public WeightedBranch(Func<float> weightCalc) => _weightCalc = weightCalc;

        /// <summary>
        /// 依据历史计算有效权重。这是原版 GetStateWeight 的忠实移植
        /// （见 RandomBranchState.cs L130-L167）。
        /// </summary>
        public float GetWeight(IReadOnlyList<string> stateLog)
        {
            float weight = 1f;

            if (RepeatType == MoveRepeatType.UseOnlyOnce)
            {
                // 本场战斗用过一次就彻底排除
                if (stateLog.Contains(StateId)) weight = 0f;
            }
            else if (RepeatType != MoveRepeatType.CanRepeatForever)
            {
                // CannotRepeat 视作 maxTimes=1；CanRepeatXTimes 用配置值
                float maxTimes = RepeatType == MoveRepeatType.CannotRepeat ? 1 : MaxTimes;

                // 历史条数不足上限时权重正常；达到上限先置 0，再看是否"连续"
                weight = stateLog.Count < maxTimes ? 1f : 0f;

                // 从末尾往前数：只要中途出现别的招，就说明不构成连续 N 次 → 恢复权重
                int checkedCount = 0;
                while (stateLog.Count >= maxTimes && checkedCount < maxTimes
                       && stateLog.Count - checkedCount > 0)
                {
                    if (stateLog[stateLog.Count - 1 - checkedCount] != StateId)
                    {
                        weight = 1f;
                        break;
                    }
                    checkedCount++;
                }
            }

            // 冷却：最近 Cooldown 个 move 中出现过本招 → 权重归零
            if (Cooldown > 0 && stateLog.Reverse().Take(Cooldown).Any(id => id == StateId))
            {
                return 0f;
            }

            return weight * BaseWeight;
        }
    }

    /// <summary>加权随机分支。对应原版 RandomBranchState。</summary>
    public sealed class RandomBranchState : MonsterState
    {
        public override string Id { get; }
        private readonly List<WeightedBranch> _branches = new();

        public override bool ShouldAppearInLogs => false;  // 分支节点不入日志

        public RandomBranchState(string stateId) => Id = stateId;

        public RandomBranchState AddBranch(WeightedBranch branch)
        {
            _branches.Add(branch);
            return this;
        }

        // 注意：原版用 Rng（可复现种子）来抽样，这里以委托形式注入随机源
        private readonly Func<float, float> _roll;

        /// <param name="roll">给定上界，返回 [0, upper) 的随机浮点；替换成你的 seeded Rng。</param>
        public RandomBranchState(string stateId, Func<float, float> roll) : this(stateId) => _roll = roll;

        public override string GetNextState(IReadOnlyList<string> stateLog)
        {
            var weights = _branches.Select(b => b.GetWeight(stateLog)).ToArray();
            float total = weights.Sum();

            // 原版这里会抛异常；复刻时给个兜底更稳（所有分支都耗尽时取第一个）
            if (total <= 0f) return _branches[0].StateId;

            float r = _roll(total);          // 原版：rng.NextFloat(total)
            for (int i = 0; i < _branches.Count; i++)
            {
                r -= weights[i];
                if (weights[i] > 0 && r <= 0f) return _branches[i].StateId;
            }
            // 浮点边界兜底
            return _branches.Last(b => b.GetWeight(stateLog) > 0).StateId;
        }
    }

    /// <summary>条件分支：取第一个条件成立的分支。对应原版 ConditionalBranchState。</summary>
    public sealed class ConditionalBranchState : MonsterState
    {
        public override string Id { get; }
        private readonly List<(Func<bool> Condition, string StateId)> _branches = new();

        public override bool ShouldAppearInLogs => false;

        public ConditionalBranchState(string stateId) => Id = stateId;

        public ConditionalBranchState AddState(string stateId, Func<bool> condition)
        {
            _branches.Add((condition, stateId));
            return this;
        }

        public override string GetNextState(IReadOnlyList<string> stateLog)
        {
            foreach (var (condition, stateId) in _branches)
                if (condition()) return stateId;
            throw new InvalidOperationException($"ConditionalBranchState '{Id}' 没有可用分支");
        }
    }

    // ------------------------------------------------------------------------
    // 4) 状态机容器：推进 / 锁首招 / 记录日志
    //    对应原版 MonsterMoveStateMachine（见其 FindNextMoveState L54-L80）
    // ------------------------------------------------------------------------
    public sealed class MonsterMoveStateMachine
    {
        private readonly Dictionary<string, MonsterState> _states = new();
        public IReadOnlyDictionary<string, MonsterState> States => _states;

        /// <summary>只记录 MoveState（分支节点 ShouldAppearInLogs=false）。</summary>
        public List<string> StateLog { get; } = new();

        private readonly MonsterState _initialState;
        private MonsterState _currentState;
        private bool _performedFirstMove;

        public MonsterMoveStateMachine(IEnumerable<MonsterState> states, MonsterState initialState)
        {
            foreach (var s in states) _states[s.Id] = s;
            _initialState = initialState;
            _currentState = initialState;
        }

        /// <summary>当前应当展示 / 执行的招式。</summary>
        public MoveState CurrentMove => (MoveState)_currentState;

        /// <summary>推进状态机，返回新一轮的招式（在玩家回合开始时调用）。</summary>
        public MoveState RollMove()
        {
            if (!_currentState.CanTransitionAway || (!_performedFirstMove && _currentState.IsMove))
                return CurrentMove;   // 首招锁定 / 必须执行过一次，不推进

            // 循环穿过所有分支节点，直到落在 MoveState 上
            MonsterState? toLog = null;
            do
            {
                string next = _currentState.GetNextState(StateLog);
                SetCurrentState(string.IsNullOrEmpty(next) ? _initialState : _states[next]);
                toLog ??= _currentState.ShouldAppearInLogs ? _currentState : null;
            }
            while (!_currentState.IsMove);

            if (toLog != null) StateLog.Add(toLog.Id);
            return CurrentMove;
        }

        /// <summary>当前招式执行完毕后调用（原版 OnMovePerformed）。</summary>
        public void OnMovePerformed(MoveState _) => _performedFirstMove = true;

        /// <summary>强制切换状态（原版 ForceCurrentState / SetMoveImmediate，用于被打断）。</summary>
        public void ForceCurrentState(MonsterState state) => SetCurrentState(state);

        private void SetCurrentState(MonsterState state)
        {
            _currentState.OnExitState();
            _currentState = state;
            _currentState.OnEnterState();
        }
    }
}
```

---

## 5. 原版真实案例（附注释）

### 案例 1：固定循环 + 随机三选一 — `SludgeSpinner`

[SludgeSpinner.cs L35-L50](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Monsters/SludgeSpinner.cs#L35-L50)，最标准的模板：

```csharp
protected override MonsterMoveStateMachine GenerateMoveStateMachine()
{
    List<MonsterState> list = new List<MonsterState>();

    // 三个招式，各带自己的意图（注意：意图只是展示，效果在下面各自的 Move 方法里）
    MoveState oilSpray = new MoveState("OIL_SPRAY_MOVE", OilSprayMove,
        new SingleAttackIntent(OilSprayDamage), new DebuffIntent());   // 攻击 + 减益
    MoveState slam = new MoveState("SLAM_MOVE", SlamMove,
        new SingleAttackIntent(SlamDamage));                           // 纯攻击
    MoveState rage = new MoveState("RAGE_MOVE", RageMove,
        new SingleAttackIntent(RageDamage), new BuffIntent());         // 攻击 + 增益

    // 三招打完都指向同一个随机分支，且都不能连续两次
    RandomBranchState rnd = new RandomBranchState("RAND");
    rnd.AddBranch(oilSpray, MoveRepeatType.CannotRepeat);
    rnd.AddBranch(slam, MoveRepeatType.CannotRepeat);
    rnd.AddBranch(rage, MoveRepeatType.CannotRepeat);

    oilSpray.FollowUpState = rnd;
    slam.FollowUpState = rnd;
    rage.FollowUpState = rnd;

    list.Add(rnd); list.Add(oilSpray); list.Add(slam); list.Add(rage);

    // initialState = oilSpray：首回合固定出 Oil Spray
    return new MonsterMoveStateMachine(list, oilSpray);
}
```

### 案例 2：按条件分流 — `Myte`

[Myte.cs L46-L62](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Monsters/Myte.cs#L46-L62)，
用 `ConditionalBranchState` 作初始状态，按站位决定第一招：

```csharp
MoveState toxic = new MoveState("TOXIC_MOVE", ToxicMove, new StatusIntent(2));      // 塞 2 张状态牌
MoveState bite = new MoveState("BITE_MOVE", BiteMove, new SingleAttackIntent(BiteDamage));
MoveState suck = new MoveState("SUCK_MOVE", SuckMove,
    new SingleAttackIntent(SuckDamage), new BuffIntent());                          // 吸血 + 自加力量

// 初始就是分支节点：不同槽位的 Myte 起手不同
ConditionalBranchState init = new ConditionalBranchState("INIT_MOVE");
init.AddState(toxic, () => base.Creature.SlotName == "first");   // 位置 1 → 塞状态牌
init.AddState(suck, () => base.Creature.SlotName == "second");   // 位置 2 → 吸血

toxic.FollowUpState = bite;
bite.FollowUpState = suck;
suck.FollowUpState = toxic;      // 回到循环起点

return new MonsterMoveStateMachine(list, init);   // ← 注意初始状态是分支节点
```

### 案例 3：多意图 + 条件突变 — `Queen`

[Queen.cs L132-L162](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Monsters/Queen.cs#L132-L162)：

```csharp
MoveState puppet = new MoveState("PUPPET_STRINGS_MOVE", PuppetStringsMove, new CardDebuffIntent());
MoveState youreMine = new MoveState("YOUR_MINE_MOVE", YoureMineMove, new DebuffIntent());
BurnBrightForMeState = new MoveState("BURN_BRIGHT_FOR_ME_MOVE", BurnBrightForMeMove,
    new BuffIntent(), new DefendIntent());                                  // 同回合两个意图
MoveState offWithHead = new MoveState("OFF_WITH_YOUR_HEAD_MOVE", OffWithYourHeadMove,
    new MultiAttackIntent(OffWithYourHeadDamage, 5));                       // 5 段攻击

// 条件分支：随从（Amalgam）死没死，决定接下来走哪招
ConditionalBranchState branch = new ConditionalBranchState("YOURE_MINE_NOW_BRANCH");
branch.AddState(BurnBrightForMeState, () => !HasAmalgamDied);
branch.AddState(offWithHead, () => HasAmalgamDied);
```

运行时可**强制换招**打断当前意图 —— 见
[Queen.cs L224-L239](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Monsters/Queen.cs#L224-L239)：

```csharp
if (base.NextMove == BurnBrightForMeState)
    SetMoveImmediate(EnragedState);   // 立即切换到 Enrage，覆盖原本要出的招
```

### 案例 4：眩晕 / 蓄力 / 必杀 — `WaterfallGiant`

[WaterfallGiant.cs L185-L216](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Monsters/WaterfallGiant.cs#L185-L216)，
展示 `MustPerformOnceBeforeTransitioning` 与 `StunIntent` / `DeathBlowIntent` 的配合：

```csharp
AboutToBlowState = new MoveState("ABOUT_TO_BLOW_MOVE", AboutToBlowMove, new StunIntent())
{
    MustPerformOnceBeforeTransitioning = true    // 蓄力回合不可被跳过（先锁住再爆炸）
};
MoveState explode = new MoveState("EXPLODE_MOVE", ExplodeMove,
    new DeathBlowIntent(() => SteamEruptionDamage));   // 数值动态取当前压力层数
```

---

## 6. 复刻落地清单（Checklist）

移植这套系统时，按下面顺序逐项确认：

- [ ] **意图只做展示**：不要用意图对象驱动伤害计算，效果必须读 move 方法体。
- [ ] **状态机推进**照抄 `FindNextMoveState`：循环穿过分支节点、首回合锁招、分支节点不入 StateLog。
- [ ] **权重计算**照抄 `GetStateWeight`：`UseOnlyOnce` 全量去重、`CannotRepeat`/`CanRepeatXTimes` 看末尾连续段、`cooldown` 看最近 N 步。
- [ ] **数值动态化**：伤害字段设计成 `int | Func<int>`，兼容"随层数/回合变化"的招式。
- [ ] **初始状态可为分支节点**：不要把 initial 硬编码成 MoveState（Myte 依赖这一点）。
- [ ] **强制换招接口**：保留 `ForceCurrentState` / `SetMoveImmediate`，用于 Boss 阶段切换与打断。
- [ ] **观测编码**：把 `IntentType` 做 one-hot，攻击类额外带 `SingleDamage` 与 `Repeats`；本项目实现见 [observation.py](file:///c:/Project/sts2-rl-agent/sts2_env/gym_env/observation.py) 的 enemies 段。

### 本项目 Python 实现对照

| 原版 (C#) | 本项目 (Python) |
|---|---|
| `AbstractIntent` + 子类 | [intents.py](file:///c:/Project/sts2-rl-agent/sts2_env/monsters/intents.py) 的 `Intent` dataclass + 工厂函数 |
| `MoveState` | [state_machine.py `MoveState`](file:///c:/Project/sts2-rl-agent/sts2_env/monsters/state_machine.py#L39-L84) |
| `RandomBranchState` / `StateWeight` | [state_machine.py `RandomBranchState` / `WeightedBranch`](file:///c:/Project/sts2-rl-agent/sts2_env/monsters/state_machine.py#L87-L176) |
| `ConditionalBranchState` | [state_machine.py `ConditionalBranchState`](file:///c:/Project/sts2-rl-agent/sts2_env/monsters/state_machine.py#L179-L195) |
| `MonsterMoveStateMachine` | [state_machine.py `MonsterAI`](file:///c:/Project/sts2-rl-agent/sts2_env/monsters/state_machine.py#L198-L264) |
| 各怪物 `GenerateMoveStateMachine()` | `sts2_env/monsters/act1.py` / `act1_weak.py` / `act2.py` / `act3.py` / `act4.py` |
