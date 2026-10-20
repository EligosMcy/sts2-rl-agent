# STS2 地图节点生成逻辑（复刻规范）

本文档完整描述《杀戮尖塔 2》**地图节点的生成逻辑与全部约束条件**，来源为反编译代码
`MegaCrit.Sts2.Core.Map`，目标是让读者可以逐位复刻（seed 级确定性）一致的地图。

> 所有结论均从反编译源码逐行核对得出，关键处标注源码位置。
> 与现有模拟器实现的差异见 [§11 现有模拟器实现对照](#11-现有模拟器实现对照)。

---

## 0. 源码索引

| 文件 | 作用 |
|------|------|
| [StandardActMap.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs) | **主线地图生成算法**（本文重点） |
| [MapPoint.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPoint.cs) | 节点结构（坐标、父子边、类型） |
| [MapCoord.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapCoord.cs) | 坐标 `(col, row)`，`CompareTo` 按 col→row 字典序 |
| [MapPointType.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPointType.cs) | 节点类型枚举 |
| [MapPointTypeCounts.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPointTypeCounts.cs) | 各类房间的默认数量池 |
| [ActMap.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/ActMap.cs) | 网格基类（含 Boss/起始点的特殊索引） |
| [MapPathPruning.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPathPruning.cs) | 重复路径剪枝（生成流程的第 4 步） |
| [MapPostProcessing.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPostProcessing.cs) | 居中 / 展开 / 拉直（第 5 步） |
| [GoldenPathActMap.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/GoldenPathActMap.cs) | 教学用固定单列地图 |
| [SpoilsActMap.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/SpoilsActMap.cs) | `SpoilsMap` 卡牌专用沙漏型地图 |
| [Rng.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Random/Rng.cs) | 种子 RNG（`System.Random` 包装） |
| [ListExtensions.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Extensions/ListExtensions.cs) | `StableShuffle` / `UnstableShuffle` |
| [ActModel.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/ActModel.cs) | 每个 Act 的房间数、类型数量入口 |
| [RunManager.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Runs/RunManager.cs#L544-L575) | 地图生成调用点与生成后修正 |

---

## 1. 数据结构与坐标系

### 1.1 节点类型枚举

[MapPointType.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPointType.cs#L3-L14)（**数值参与剪枝哈希，不能改动顺序**）：

```
Unassigned = 0   // 尚未赋型的占位
Unknown    = 1   // "?" 房间，运行时再 roll
Shop       = 2
Treasure   = 3
RestSite   = 4
Monster    = 5
Elite      = 6
Boss       = 7
Ancient    = 8   // 起始点（Neow 等）
```

### 1.2 节点对象

[MapPoint.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPoint.cs#L8-L46)：

- `coord: MapCoord(col, row)`
- `parents: HashSet<MapPoint>`、`Children: HashSet<MapPoint>` — **引用语义集合**，不是坐标比较。
  注意 `MapPoint` 只有 `private bool Equals(MapPoint)`（私有方法），并未 override `object.Equals`/`GetHashCode`，
  因此集合去重、`Contains`、`object.Equals` 全部按**对象引用**判断。
- `PointType`、`CanBeModified`（默认 `true`）
- `AddChildPoint(child)` 同时双向登记 child 与 parent。

### 1.3 网格与坐标语义

`Grid = new MapPoint[7, _mapLength]`（[StandardActMap.cs#L74](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs#L74)）：

- **列 `col`：0..6（共 7 列）**
- **行 `row`：0..`_mapLength-1` 为网格内行；Boss/起始点不在网格里**
- `_mapLength = act.GetNumberOfRooms(isMultiplayer) + 1`
  （[ActModel.GetNumberOfRooms](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/ActModel.cs#L175-L183) = `BaseNumberOfRooms - (多人 ? 1 : 0)`）
- `StartingMapPoint = (3, 0)`，`BossMapPoint = (3, GetRowCount())`，即 **Boss 行 = `_mapLength`**
- 若 `hasSecondBoss`：`SecondBossMapPoint = (3, GetRowCount() + 1)`
- `GetPoint(col,row)` 会特判返回 Boss / 第二 Boss / 起始点，因此它们虽不在 `Grid` 里也能被查询到。
- **房间行是 `1 .. _mapLength-1`**，第 0 行只属于起始点。

`ActMap.GetPoint`、`HasPoint`、`IsInMap` 均按上述特判实现（[ActMap.cs#L58-L122](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/ActMap.cs#L58-L122)）。

### 1.4 RNG 与洗牌语义

- [Rng.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Random/Rng.cs#L18-L29)：
  `new Rng(seed)` → `System.Random((int)seed)`；`new Rng(seed, name)` → `seed + (uint)StringHelper.GetDeterministicHashCode(name)`。
- 地图 RNG：`new Rng(runState.Rng.Seed, $"act_{CurrentActIndex + 1}_map")`
  （[StandardActMap.CreateFor](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs#L94-L97)）
- `Rng.Counter` 只在 `NextInt/NextBool/NextFloat/NextDouble/NextUnsignedInt` 等递增；
  **`NextGaussianInt` 与 `NextGaussianDouble` 内部直接用 `_random.NextDouble()`，`Counter` 不变**
  （[Rng.cs#L141-L154](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Random/Rng.cs#L141-L154)）。
  复刻时必须照抄这一"计数不增但消耗随机流"的行为，否则 `Counter` 对不上。
- `NextGaussianInt(mean, stdDev, min, max)`：**拒绝采样**（`u1 = 1 - rand; u2 = 1 - rand; z = sqrt(-2 ln u1) * sin(2π u2); round(mean + stdDev*z)`，
  不在 `[min,max]` 就重抽），不是 clamp。
- `UnstableShuffle(list)`：Fisher–Yates，`for (i = n-1; i > 0; i--) swap(i, NextInt(i+1))`。
  对长度 n 的列表恰好消耗 **n-1 次** `NextInt`。
- `StableShuffle(list)`：先 `Sort()`（默认比较器）再 `UnstableShuffle`
  （[ListExtensions.cs#L10-L19](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Extensions/ListExtensions.cs#L10-L19)）。
  - 对 `List<int>`、`List<MapPointType>`：按数值升序。
  - 对 `List<MapPoint>`：按 `MapCoord.CompareTo`，即 **先 col 后 row 的字典序**。
  - **这是复刻中极易漏掉的一步**：不做 `Sort` 直接 shuffle 会得到不同的结果。

---

## 2. 生成总流程

构造函数 [StandardActMap.cs#L70-L92](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs#L70-L92) 的执行顺序（**顺序本身影响 RNG 结果**）：

```
1. _mapLength = act.GetNumberOfRooms(isMultiplayer) + 1
2. Grid = new MapPoint[7, _mapLength]
3. _rng = mapRng
4. _pointTypeCounts = act.GetMapPointTypes(mapRng)      <-- ① 房间数量 roll（先于路径生成！）
5. 建立 BossMapPoint(3, _mapLength) / StartingMapPoint(3, 0) / [SecondBossMapPoint(3, _mapLength+1)]
6. GenerateMap()                                        <-- ② 7 条随机路径
7. AssignPointTypes()                                   <-- ③ 固定行 + 池赋型
8. MapPathPruning.PruneDuplicateSegments(...)           <-- ④ 剪枝（可用 enablePruning=false 关闭）
9. Grid = MapPostProcessing.CenterGrid(Grid)            <-- ⑤ 居中
10. Grid = MapPostProcessing.SpreadAdjacentMapPoints(Grid) <-- ⑥ 同层展开
11. Grid = MapPostProcessing.StraightenPaths(Grid)      <-- ⑦ 拉直
```

调用入口：`RunManager.GenerateMap()` → `State.Act.CreateMap(State, replaceTreasureWithElites: false)`
（[RunManager.cs#L544-L575](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Runs/RunManager.cs#L544-L575)、
[ActModel.CreateMap](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/ActModel.cs#L409-L412)）。

---

## 3. 步骤 ①：房间数量池（GetMapPointTypes）

### 3.1 默认池

[MapPointTypeCounts.cs#L26-L32](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPointTypeCounts.cs#L26-L32)：

```
NumOfElites   = round(5 * (Ascension >= SwarmingElites(=1) ? 1.6 : 1.0))   // A0 = 5，A1+ = 8
NumOfShops    = 3
NumOfUnknowns = rng.NextGaussianInt(mean=12, stdDev=1, min=10, max=14)
NumOfRests    = rng.NextGaussianInt(mean=5,  stdDev=1, min=3,  max=6)
```

`PointTypesThatIgnoreRules` 默认空集合；若某类型在其中，则 §6 的所有约束对它全部跳过
（[MapPointTypeCounts.cs#L21-L24](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPointTypeCounts.cs#L21-L24)、
[StandardActMap.cs#L356-L359](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs#L356-L359)）。原版各 Act 均未使用该字段，做 mod 时需要留意。

### 3.2 各 Act 的覆写与**精确 RNG 调用序列**

| Act | BaseNumberOfRooms | 覆写 | RNG 调用顺序（同一 mapRng） |
|-----|------------------|------|---------------------------|
| Overgrowth | 15 | `NumOfRests = NextGaussianInt(7,1,6,7)` | ① gauss(7,1,6,7) → ② gauss(12,1,10,14) → ③ gauss(5,1,3,6)（③ 的结果被丢弃） |
| Underdocks | 15 | 同 Overgrowth | 同 Overgrowth |
| Hive | 14 | `NumOfUnknowns = 副本.NumOfUnknowns - 1`，`NumOfRests = NextGaussianInt(6,1,6,7)` | 先在**副本 RNG** 上 gauss(12,1,10,14)、gauss(5,1,3,6)；再在原 RNG 上 gauss(**6**,1,6,7) → gauss(12,1,10,14) → gauss(5,1,3,6) |
| Glory | 13 | `NumOfUnknowns = 副本.NumOfUnknowns - 1`，`NumOfRests = NextInt(5,7)`（即 5 或 6） | 副本：gauss(12,1,10,14)、gauss(5,1,3,6)；原 RNG：`NextInt(5,7)` → gauss(12,1,10,14) → gauss(5,1,3,6) |

源码：[Overgrowth.cs#L127-L138](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Acts/Overgrowth.cs#L127-L138)、
[Underdocks.cs#L105-L116](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Acts/Underdocks.cs#L105-L116)、
[Hive.cs#L111-L125](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Acts/Hive.cs#L111-L125)、
[Glory.cs#L99-L113](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Acts/Glory.cs#L99-L113)。

关键坑：

- 返回语句是 `new MapPointTypeCounts(mapRng) { NumOfRests = ... }`，即**构造函数仍会在同一 mapRng 上消耗 2 次高斯**，
  只有返回值被覆盖，随机流照常推进。
- Hive / Glory 的"副本 RNG"是 `new Rng(mapRng.Seed, mapRng.Counter)`，用来只取 `NumOfUnknowns` 而不推进主流。
  此写法在 `Counter != 0` 时并**不能**真正复制 `System.Random` 状态（`FastForwardCounter` 只推进 `Next()`），
  但此处地图 RNG 刚创建、`Counter == 0`，所以副本状态与主流一致。
- 顺序上 ① 在路径生成之前，见 §2。

---

## 4. 步骤 ②：路径生成（GenerateMap / PathGenerate）

[StandardActMap.cs#L190-L217](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs#L190-L217)

### 4.1 生成 7 条路径

```
for i in 0..6:                                  # _iterations = 7
    start = GetOrCreatePoint(NextInt(0,7), row=1)   # col ∈ 0..6
    if i == 1:                                     # 仅第 2 条路径
        while start in startMapPoints:              # 必须换一个起点（会重复 roll 直到不同）
            start = GetOrCreatePoint(NextInt(0,7), row=1)
    startMapPoints.add(start)
    PathGenerate(start)
```

注意 `NextInt(0,7)` 是 C# `NextInt(minInclusive, maxExclusive)` ⇒ **`0..6`，永不越界**。

### 4.2 单条路径推进

[PathGenerate](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs#L116-L126)：

```
cur = start
while cur.row < _mapLength - 1:        # 顶部房间行是 _mapLength-1
    next = GetOrCreatePoint(GenerateNextCoord(cur))
    cur.AddChildPoint(next)
    cur = next
```

### 4.3 下一步坐标

[GenerateNextCoord](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs#L128-L165)：

```
col = cur.col
minCol = max(0, col-1)
maxCol = min(col+1, 6)
dirs = [-1, 0, 1]
StableShuffle(dirs, rng)                 # 长度 3 ⇒ 消耗 NextInt(3)、NextInt(2) 共 2 次 roll
for d in dirs:
    targetCol = d == -1 ? minCol : (d == 0 ? col : maxCol)
    if not HasInvalidCrossover(cur, targetCol):
        return (targetCol, cur.row + 1)
throw InvalidOperationException          # 三条方向全被否决时抛异常（理论上不会发生）
```

### 4.4 交叉检测

[HasInvalidCrossover](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs#L167-L188)：

```
d = targetCol - cur.col
if d == 0: return false                  # 直线上升永不交叉
sameRowPoint = Grid[targetCol, cur.row]
if sameRowPoint == null: return false
for child in sameRowPoint.Children:
    if (child.col - sameRowPoint.col) == -d: return true
return false
```

即：**对角线方向与同层相邻节点的已有子边方向相反时，视为交叉而拒绝**。
这是保证"边不相交"的唯一机制。

### 4.5 首尾连接

```
ForEachInRow(row = _mapLength-1): p.AddChildPoint(BossMapPoint)
if SecondBossMapPoint != null: BossMapPoint.AddChildPoint(SecondBossMapPoint)
ForEachInRow(row = 1): StartingMapPoint.AddChildPoint(p)
```

---

## 5. 步骤 ③-A：固定行赋型

[AssignPointTypes](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs#L231-L289)，顺序固定：

| 顺序 | 行 | 条件 | 类型 | `CanBeModified` |
|------|----|------|------|-----------------|
| 1 | `GetRowCount()-1`（= `_mapLength-1`，顶部房间行） | 总是 | `RestSite` | `false` |
| 2 | `GetRowCount()-7`（= `_mapLength-7`） | `ShouldReplaceTreasureWithElites == false` | `Treasure` | `false` |
| 2' | 同上 | `ShouldReplaceTreasureWithElites == true` | `Elite` | `false` |
| 3 | `1`（最底层房间行） | 总是 | `Monster` | 不变（默认 `true`） |
| 4 | 池赋型（见 §6） | — | — | — |
| 5 | 剩余 `Unassigned` | 总是 | `Monster` | — |
| 6 | `BossMapPoint` | 总是 | `Boss` | — |
| 7 | `StartingMapPoint` | 总是 | `Ancient` | — |
| 8 | `SecondBossMapPoint` | 存在时 | `Boss` | — |

**注意第 6/7 步在池赋型之后**：`Boss`/`Ancient` 不参与 `Unassigned → Monster` 的兜底。
`ShouldReplaceTreasureWithElites` 由 `RunManager` 传入（普通局恒为 `false`，由 mod/`BigGameHunter` 之类改写，见 §9.3）。

> `StandardActMap` 中还有 `EnsureRowsContainsPointType` / `AssignPointTypesToRandomRows` / `GetRows` 三个**私有方法未被任何地方调用**（死代码）。
> 它们**不参与**原版生成流程，复刻时不要实现。

---

## 6. 步骤 ③-B：随机池赋型与 5 条约束

### 6.1 池与队列

```
pool = Queue<MapPointType>( [RestSite] * NumOfRests
                          + [Shop]     * NumOfShops
                          + [Elite]    * NumOfElites
                          + [Unknown]  * NumOfUnknowns )     # 注意拼接顺序
points = GetAllMapPoints().ToList()        # 只含 Grid 内节点（列优先：col 0..6 外层，row 0.._mapLength-1 内层）
                                           # Start/Boss 不在 Grid 中，因此不参与本次赋型
StableShuffle(points, rng)                 # 先按 (col,row) 排序，再 Fisher–Yates
for p in points:
    if p.PointType == Unassigned:
        p.PointType = GetNextValidPointType(pool, p)
```

[AssignRemainingTypesToRandomPoints](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs#L341-L349)
（`GetAllMapPoints` 的遍历顺序见 [ActMap.cs#L27-L40](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/ActMap.cs#L27-L40)）。

### 6.2 取型（循环队列）

[GetNextValidPointType](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs#L351-L367)：

```
for i in 0..pool.Count-1:          # 最多转一圈
    t = pool.Dequeue()
    if PointTypesThatIgnoreRules.Contains(t): return t
    if IsValidPointType(t, p):            return t     # 命中即"消费"该类型（不再入队）
    pool.Enqueue(t)                                    # 否则放回队尾
return Unassigned                                  # 一圈都不合法 ⇒ 交给 §5 第 5 步兜底为 Monster
```

### 6.3 五条约束（全部通过才合法）

[IsValidPointType](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs#L369-L392)，限制集合定义于
[L27-L58](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs#L27-L58)：

| # | 规则 | 受限类型 | 判定 |
|---|------|---------|------|
| 1 | **下层限制**：`row < 5` | `{RestSite, Elite}` | 该行不得出现这些类型 |
| 2 | **上层限制**：`row >= _mapLength - 3` | `{RestSite}` | 顶部 3 行内不得出现休息点 |
| 3 | **父/子邻接** | `{Elite, RestSite, Treasure, Shop}` | `parents ∪ Children` 中不得存在同类型（注意：**同时检查 parents 和 Children**，两行代码语义重复） |
| 4 | **子邻接** | `{Elite, RestSite, Treasure, Shop}` | `Children` 中不得存在同类型 |
| 5 | **兄弟唯一** | `{RestSite, Monster, Unknown, Elite, Shop}` | 兄弟节点（`parents.SelectMany(x => x.Children)` 去掉自身）中不得存在同类型 |

判定顺序：规则 1 → 2 → 3 → 4 → 5，任一失败即返回 `false`（顺序不影响结果，但影响调试输出）。

> 规则 1 的 `row < 5` 是**硬编码常量 5**，与 `_mapLength` 无关。
> 规则 5 的兄弟定义只沿"父→子"一层展开，**不含自身**（用引用比较 `object.Equals`）。

---

## 7. 步骤 ④：重复路径剪枝（MapPathPruning）

`enablePruning` 默认为 `true`。**剪枝发生在赋型之后**（段 key 里含 `PointType`），也在后处理之前。
入口 [PruneDuplicateSegments](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPathPruning.cs#L13-L26)：

```
n = 0
segments = FindMatchingSegments(StartingMapPoint)
while PrunePaths(grid, startMapPoints, segments, rng):
    n += 1
    if n > 50: throw InvalidOperationException   # 循环上限 50
    segments = FindMatchingSegments(StartingMapPoint)
```

### 7.1 枚举所有路径与段（FindMatchingSegments）

[FindAllPaths](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPathPruning.cs#L39-L70)：
从 `StartingMapPoint` 递归收集所有到 `Boss` 的路径（`PointType == Boss` 时终止）。

[AddSegmentsToDictionary](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPathPruning.cs#L72-L104) 对每条 path：

```
for i in 0..path.Count-2:
    if not IsValidSegmentStart(path[i]): continue
        # IsValidSegmentStart: path[i].Children.Count > 1  或者  path[i].coord.row == 0
    for j in 2 .. (path.Count-1-i):                 # 段长至少 3 个节点
        end = path[i+j]
        if not IsValidSegmentEnd(end): continue      # 要求 end.parents.Count >= 2
        seg = path[i .. i+j]
        key = GenerateSegmentKey(seg)
        if key 不存在: segments[key] = [seg]
        elif 与已有段 AnyOverlappingSegments 为 false: segments[key].add(seg)
```

段 key [GenerateSegmentKey](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPathPruning.cs#L120-L155)：

```
起点 row == 0  ->  "{0}-{endCol},{endRow}-" + join(",", seg.map(t => (int)t.PointType))
否则           ->  "{startCol},{startRow}-{endCol},{endRow}-" + join(",", seg.map(t => (int)t.PointType))
```

[OverlappingSegment](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPathPruning.cs#L162-L176)：
两段长度都 ≥ 3，且存在 `i ∈ [1, len-2]` 使 `a[i]` 与 `b[i]` 是**同一对象引用** ⇒ 视为重叠。

重复段集合 = 段数 > 1 的 key 分组（[GetDuplicateSegments](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPathPruning.cs#L178-L181)）。

### 7.2 剪枝动作

[PrunePaths](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPathPruning.cs#L183-L198)：遍历每个重复段分组（**顺序为字典 Values 顺序**）：

```
UnstableShuffle(group, rng)
if PruneAllButLast(grid, startMapPoints, group) != 0: return true   # 保留了"最后一个"（shuffle 后）
if BreakAParentChildRelationshipInAnySegment(group):   return true
# 无变化则继续下一个分组
return false
```

- [PruneAllButLast](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPathPruning.cs#L200-L215)：对除**最后一个**以外的每个段执行 `PruneSegment`，返回成功次数。
- [PruneSegment](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPathPruning.cs#L217-L246)：

```
for i in 0..seg.Length-2:
    p = seg[i]
    if not IsInMap(grid, p): return true        # 已不在图上（例外：Ancient/Boss 视为在图上）
    if p.Children.Count > 1 or p.parents.Count > 1
       or p.parents.Any(n => n.Children.Count == 1 and not IsRemoved(grid, n)):
        continue                                # 关键节点，跳过
    tail = seg[i..]
    if tail.Any(n => n.Children.Count > 1 and n.parents.Count == 1): continue
    if seg[^1].parents.Count == 1: return false
    if not p.Children.Where(c => not seg.Contains(c)).Any(c => c.parents.Count == 1):
        RemovePoint(grid, startMapPoints, p); result = true
```

- [RemovePoint](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPathPruning.cs#L248-L260)：
  `grid[col,row] = null`，从 `startMapPoints` 移除，并断开全部亲子边。
- [BreakAParentChildRelationshipInSegment](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPathPruning.cs#L288-L305)：
  若 `seg[i].Children.Count >= 2` 且 `seg[i+1].parents.Count != 1`（即该子节点仍有其它父），则删除 `seg[i] → seg[i+1]` 这条边。

> 复刻要点：剪枝消费 `_rng`（`UnstableShuffle(group)`，长度 k 的分组消耗 k-1 次 `NextInt`），
> 且每轮结束后需要**重新枚举**所有段并再次循环，直到某一轮完全没有改动。上限 50 轮。

---

## 8. 步骤 ⑤⑥⑦：后处理（MapPostProcessing）

后处理**只改 `coord.col` 与网格位置，不改父子关系**，但会改变后续（跨 act 的）坐标记录。
顺序固定为 CenterGrid → SpreadAdjacentMapPoints → StraightenPaths。

### 8.1 CenterGrid

[MapPostProcessing.cs#L9-L69](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPostProcessing.cs#L9-L69)

```
leftEmpty  = 第 0 列与第 1 列都为空
rightEmpty = 第 (C-1) 列与第 (C-2) 列都为空
shift = leftEmpty && !rightEmpty ? -1
      : !leftEmpty && rightEmpty ? +1
      : 0
if shift == 0: 原样返回
shift > 0: 从右往左扫描每列搬移（越界的目标格点被丢弃）
shift < 0: 从左往右扫描每列搬移
```

（`shift > 0` 时只可能丢弃空列，因为触发条件已保证最右两列为空。）

### 8.2 SpreadAdjacentMapPoints

[MapPostProcessing.cs#L155-L205](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPostProcessing.cs#L155-L205)，**逐行独立处理**：

```
for row in 0..rows-1:
    nodes = 该行所有非空节点
    repeat until 无移动:
        for n in nodes:
            allowed = 所有列 ∩ (每个 parent.col 的 {col-1,col,col+1}) ∩ (每个 child.col 的 {col-1,col,col+1})
            curGap = 到同层其它节点的最小列距（本行只有自己时为 int.MaxValue）
            best = n.col, bestGap = curGap
            for cand in allowed:                      # 注意：遍历的是 HashSet<int>，顺序不保证
                if cand != n.col and (grid[cand,row] == null or grid[cand,row] == n):
                    gap = ComputeGap(cand, nodes, n)
                    if gap > bestGap: best = cand; bestGap = gap
            if best != n.col:
                grid[n.col,row] = null; grid[best,row] = n; n.col = best; 标记有移动
```

- `ComputeGap` 是**到同层其它节点的最小列距**；只有严格变大才移动。
- `allowed` 用 `HashSet<int>` 迭代 ⇒ 候选顺序不确定，但在"严格更大"的判据下结果唯一（除并列情况）。
- 无父无子时 `allowed` 为全部列；父列与子列距离 > 2 时 `allowed` 可能为空（此时不移动）。

### 8.3 StraightenPaths

[MapPostProcessing.cs#L71-L112](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPostProcessing.cs#L71-L112)

```
for row, for col:
    p = grid[col,row]
    if p == null or p.parents.Count != 1 or p.Children.Count != 1: continue
    parent = p.parents.First(); child = p.Children.First()
    leftZig  = p.col < child.col and p.col < parent.col      # 左右两边都更高 ⇒ 往右挪一格
    rightZig = p.col > child.col and p.col > parent.col      # 往左挪一格
    if leftZig  and col < C-1 and grid[col+1,row] == null: 移到 col+1
    if rightZig and col > 0   and grid[col-1,row] == null: 移到 col-1
```

即"之字形"的单一通路节点被拉直；目标格被占用则不动。

---

## 9. 特殊地图与生成后修正

### 9.1 GoldenPathActMap（教学）

[GoldenPathActMap.cs#L9-L68](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/GoldenPathActMap.cs#L9-L68)：
固定单列（col = 3），16 个类型序列

```
Monster, Unknown, Monster, RestSite, Monster, RestSite, Unknown, Treasure,
Unknown, Treasure, Unknown, Shop, Elite, RestSite, Elite, RestSite
```

多人游戏时 `RemoveAt(2)` 去掉第 3 个元素（`Monster`），行数随之为 15。
无 `generate_act_map` 参与，`Grid = [7, list.Count + 1]`，`Boss`/`Ancient` 直接赋型。

### 9.2 SpoilsActMap（`SpoilsMap` 卡）

[SpoilsActMap.cs#L64-L78](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/SpoilsActMap.cs#L64-L78)，
由 [SpoilsMap.ModifyGeneratedMap](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models.Cards/SpoilsMap.cs#L49-L61) 替换当前 act 地图：

与 `StandardActMap` 的差异：

1. RNG 为 `new Rng(runState.Rng.Seed, "spoils_map")`（**不同流名**）。
2. `_treasureRow = GetRowCount() - 7`，先 `GenerateHourglassMap()` 再 `AssignPointTypes()` 再剪枝；
   **不做 CenterGrid / Spread / Straighten**。
3. `GenerateHourglassMap` 里 7 条路径的下一步坐标受"沙漏"约束
   （[GenerateNextCoord](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/SpoilsActMap.cs#L141-L202)、
   [GetAllowedColumnsForRow](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/SpoilsActMap.cs#L240-L250)）：

```
allowed(row) = 以 col=3 为中心，半径 r = min(3, min(|row - _treasureRow|, min(3, max(0, _mapLength-1-row)+1)))
               minCol = max(0, 3-r)，maxCol = min(6, 3+r)
方向优先级：到宝藏行距离 > 3        -> StableShuffle([-1,0,1])
            0 < 距离 <= 3          -> 朝中心优先列表 BuildCenteredPriorityList(col, 3)
            距离 <= 0（已过宝藏行） -> StableShuffle([-1,0,1])
另需满足：目标节点 parents.Count < 3（或已是父）、当前节点 children.Count < 3（或已是父）等额外父子容量限制；
步长 |Δcol| > 1 时直接抛异常。
```

4. 宝藏行被强制收敛为**唯一**节点 `(3, _treasureRow)`，其余同伴节点通过
   [RedirectToTreasure](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/SpoilsActMap.cs#L275-L288)
   把父边/子边全部改接到宝藏点并从网格删除。
5. `AssignPointTypes` 只固定"顶部行 = RestSite"和"第 1 行 → Monster"，
   **没有** Treasure 行固定（宝藏行已单独处理）。
6. `AssignRemainingTypesToRandomPoints` 会排除 `BossMapPoint` 与 `StartingMapPoint`，
   且 `GetNextValidPointType` 在一圈内无合法类型时返回 `Unassigned`（保持未赋值），
   之后再统一兜底为 `Monster`。约束集合与 §6.3 完全一致。

### 9.3 生成后修正（RunManager）

[RunManager.cs#L544-L575](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Runs/RunManager.cs#L544-L575)：

```
- 若 SavedMapsToLoad 中有本 act 的存档地图 -> 用 SavedActMap（不重新生成）
- 否则:
    map = Act.CreateMap(State, replaceTreasureWithElites: false)
    map = Hook.ModifyGeneratedMap(State, map, actIndex)
    await Hook.AfterMapGenerated(State, map, actIndex)
    if not State.ExtraFields.StartedWithNeow and CurrentActIndex == 0:
        map.StartingMapPoint.PointType = Monster     # 未经过 Neow 时，起点行变成 Monster 而非 Ancient
```

- `replaceTreasureWithElites` 在普通流程恒为 `false`；`BigGameHunter` 等修改器会走
  `ModifyGeneratedMap` 重新生成地图（见 [PARITY_GAPS.md](file:///c:/Project/sts2-rl-agent/docs/PARITY_GAPS.md)）。
- `SpoilsMap.ModifyGeneratedMapLate` 会把宝藏点记录为 quest 目标（`SpoilsCoord`）。
- `hasSecondBoss`（A10 `DoubleBoss`）时额外存在 `SecondBossMapPoint = (3, _mapLength+1)`，
  `BossMapPoint` 指向它；且初始行/顶部行连接逻辑与单 Boss 相同。

### 9.4 节点 → 房间的运行时映射（非生成期）

生成期只决定 `MapPointType`；进入节点时才 roll 实际房间
（`Unknown` 走 `Odds.UnknownMapPoint`，`Ancient` → Event 等）。详见
[GAME_SYSTEMS_REFERENCE.md §2.3](file:///c:/Project/sts2-rl-agent/docs/GAME_SYSTEMS_REFERENCE.md)。

---

## 10. 复刻检查清单（按 RNG 与顺序）

1. **顺序**：`GetMapPointTypes`（房间数量）→ 7 条路径 → 固定行赋型 → 队列池赋型 → 剪枝 → 居中 → 展开 → 拉直。
2. **`NextInt(min, max)` 语义**：C# 是**上界开区间**。`NextInt(0,7)` ⇒ 0..6。
3. **`StableShuffle` = 先 `Sort` 再 Fisher–Yates**。`List<MapPoint>` 排序键是 `(col, row)`。
4. **`GetAllMapPoints` 遍历顺序**：col 外层（0→6）、row 内层，**先收集再排序再洗**。
5. **`NextGaussianInt` 不推进 `Counter`**，但消耗 `_random`；拒绝采样而非 clamp。
6. **池的拼接顺序**：RestSite → Shop → Elite → Unknown，且 `Queue` 是循环消费（不合法就放回队尾）。
7. **约束 1 的 `row < 5` 是常量**；**约束 3 同时检查 parents 与 Children**；**约束 5 的兄弟只展开一层**。
8. **`Boss`/`Ancient` 在池赋型之后赋值**，因此不会被兜底成 `Monster`；`Unassigned` 兜底为 `Monster`。
9. **剪枝在赋型之后**（key 含 `PointType` 与段起终点坐标），且每轮重新枚举、最多 50 轮；剪枝会消耗 RNG。
10. **后处理只改 col**，不改父子关系；`SpreadAdjacentMapPoints` 是逐行、可能多轮迭代。
11. **`MapPoint` 的父子集合是引用语义**，不要用坐标做去重/比较。
12. **多人游戏**：`_mapLength = Base - 1`（房间少 1 行），`GoldenPath` 少 1 个节点。
13. `StandardActMap` 中的 `EnsureRowsContainsPointType` / `AssignPointTypesToRandomRows` / `GetRows` 是**死代码**，不参与生成。

---

## 11. 现有模拟器实现对照

对应实现：[sts2_env/map/generator.py](file:///c:/Project/sts2-rl-agent/sts2_env/map/generator.py)，
测试：[tests/test_map_generation_layout_parity.py](file:///c:/Project/sts2-rl-agent/tests/test_map_generation_layout_parity.py)。

已对齐的部分：网格尺寸、7 条路径、交叉检测、起点不重复、首尾连接、固定行赋型、
5 条约束、循环队列取型、每种 Act 的数量池（含 Act 0/2 的 RNG 消耗顺序）、`CenterGrid`、`SpoilsActMap` 沙漏路径。

**尚未对齐或存在差异的部分（复刻时需注意）：**

| # | 项 | 原版（C#） | 当前实现 |
|---|----|-----------|---------|
| 1 | 数量池 vs 路径生成顺序 | `GetMapPointTypes` 在路径生成**之前**（[L76 先于 L83](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs#L70-L92)） | [先跑 7 条路径再 roll 数量](file:///c:/Project/sts2-rl-agent/sts2_env/map/generator.py#L588-L617)，同 seed 会得到不同地图 |
| 2 | 剪枝 | `MapPathPruning.PruneDuplicateSegments`（默认开启） | **未实现** |
| 3 | 后处理 | CenterGrid + SpreadAdjacentMapPoints + StraightenPaths | 仅有 `_center_grid` |
| 4 | 赋型遍历顺序 | `GetAllMapPoints()` 列优先 + `StableShuffle`（先按 `(col,row)` 排序） | 取 `dict.values()` 插入序后直接 `shuffle`，[未排序](file:///c:/Project/sts2-rl-agent/sts2_env/map/generator.py#L239-L243) |
| 5 | Hive 休息点数量 | `NextGaussianInt(6, 1, 6, 7)`（mean=6） | [act_index==1 时用 mean=7](file:///c:/Project/sts2-rl-agent/sts2_env/map/generator.py#L323-L328) |
| 6 | 第二 Boss | `SecondBossMapPoint`（A10 DoubleBoss） | 未建模 |
| 7 | Neow 修正 | 未经过 Neow 时 act0 起点变 `Monster` | 未建模 |
| 8 | 搜索失败行为 | `GenerateNextCoord` 三条方向全否决时抛异常 | 回退为"直线上行" |
| 9 | Underdocks | 独立 Act（Base 15，同 Overgrowth 的 roll） | 未建模（`ALL_ACTS` 仅 0/1/2） |
| 10 | `PointTypesThatIgnoreRules` | 支持按类型跳过全部约束 | 未建模（原版各 Act 未使用） |

> 其中 #1、#4、#5 会直接影响**同 seed 的确定性复现**，若要严格对齐建议优先处理。

---

## 12. 参考

- [GAME_SYSTEMS_REFERENCE.md §1 地图生成](file:///c:/Project/sts2-rl-agent/docs/GAME_SYSTEMS_REFERENCE.md)（简要版）
- [DECOMPILATION_GUIDE.md](file:///c:/Project/sts2-rl-agent/docs/DECOMPILATION_GUIDE.md)（反编译源码导航）
- [PARITY_GAPS.md](file:///c:/Project/sts2-rl-agent/docs/PARITY_GAPS.md)（整体 parity 缺口）
