# 地图节点配置文件（Unity ScriptableObject）实现指南

> 本文档只讲**数据如何实现**。绘制/表现逻辑不在范围内（第一部分末尾会附一句 STS2 的绘制方式，仅用于理解坐标含义）。
> 结论先行：**可以**用 ScriptableObject 实现，但不能照着"节点引用节点"的直觉写——连线必须用**坐标**做 ID。

---

## 一、STS2 的地图是怎么实现的（源码结论）

源码位置：[decompiled/MegaCrit.Sts2.Core.Map/](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map)

### 1.1 数据结构总览

| 类型 | 职责 | 关键字段 | 源码 |
| --- | --- | --- | --- |
| `MapCoord` (struct) | 坐标 = **列 col + 层 row** | `int col`（列，0..6）、`int row`（层，0 起） | [MapCoord.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapCoord.cs) |
| `MapPointType` (enum) | 节点类型 | Unassigned / Unknown / Shop / Treasure / RestSite / Monster / Elite / Boss / Ancient | [MapPointType.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPointType.cs) |
| `MapPoint` | 单个节点 + **连线** | `coord`、`PointType`、`CanBeModified`、`parents: HashSet<MapPoint>`、`Children: HashSet<MapPoint>` | [MapPoint.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPoint.cs) |
| `ActMap` (abstract) | 整张地图容器 | `MapPoint?[,] Grid`（**列 × 层 二维数组**）、`startMapPoints`、`BossMapPoint`、`StartingMapPoint`、`SecondBossMapPoint` | [ActMap.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/ActMap.cs) |
| `StandardActMap` | 随机生成（STS2 实际用的） | 宽 7 列，长度 = `GetNumberOfRooms()+1` | [StandardActMap.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/StandardActMap.cs) |
| `SavedActMap` | 从存档重建 | 依赖 `SerializableActMap` | [SavedActMap.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/SavedActMap.cs) |
| `MapPointTypeCounts` | 类型**配额**配置 | `NumOfElites / NumOfShops / NumOfUnknowns / NumOfRests` | [MapPointTypeCounts.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPointTypeCounts.cs) |
| `MapPointState` (enum) | **运行时**状态，不是配置 | None / Travelable / Traveled / Untravelable | [MapPointState.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapPointState.cs) |

### 1.2 层数怎么表达

- **层 = `row`**，列 = `col`。整张图是一个 `MapPoint[列, 层]` 二维数组（`Grid[col, row]`）。
- 层数 = `Grid.GetLength(1)`。起点固定 `row = 0`（`(col=7/2, row=0)`）；Boss 在 `row = RowCount`（数组外，单独持有）；双 Boss 在 `RowCount + 1`。
- 「第几层」就是 `point.coord.row`；`ActModel.GetNumberOfFloors()` = 房间数 + 2（含起点与多 Boss 层）。
- 第 1 层有 7 个候选起点（`startMapPoints`），第 0 层的唯一 Ancient 节点与它们全部相连。

### 1.3 节点类型（重要差异）

STS2 原生 `MapPointType` 只有 9 个值。你列的 8 种节点里有 2 种 STS2 **没有独立节点类型**：

| 你的节点 | STS2 对应 | 说明 |
| --- | --- | --- |
| 商店 | `Shop` | 直接对应 |
| 事件 | `Unknown` | 图标是 `map_unknown`（问号） |
| 普通弱怪 | `Monster` | **不是节点类型**。弱怪是"房间内容"层面的划分：一个 Act 内前 `NumberOfWeakEncounters(=3)` 个怪物房从弱遭遇池抽取，之后从普通遭遇池抽取（见 [ActModel.GenerateRooms](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/ActModel.cs#L242-L264)） |
| 普通怪 | `Monster` | 直接对应 |
| 腐化普通怪 | `Monster` | 无独立类型，需你自己扩展 |
| 精英怪 | `Elite` | 直接对应 |
| 腐化精英怪 | `Elite` | 无独立类型。STS2 的"精英变多"是 Ascension `SwarmingElites`（精英数量 ×1.6），不是新节点类型 |
| Boss | `Boss` | 直接对应 |
| （起点） | `Ancient` | 建议保留，否则缺少 `row=0` 的根节点 |

结论：你要么在枚举里**自定义**这 8～9 个值（推荐，本文就这么做），要么保留 STS2 原枚举 + 额外字段标记"弱/腐化"。前者更简单。

### 1.4 连线怎么实现

**没有独立的"线"对象。** 连线 = 节点之间的父子引用：

```csharp
// MapPoint.cs
public readonly HashSet<MapPoint> parents;
public HashSet<MapPoint> Children { get; }

public void AddChildPoint(MapPoint child) {
    Children.Add(child);
    child.parents.Add(this);   // 双向同时维护
}
```

- 绘制时遍历 `point.Children` 画线：`NMapScreen.DrawPaths()` 对每个 child 端点调 `CreatePath()`，沿两点之间每 22px 摆一个 `map_dot.tscn` 小圆点（所以线其实是"点串"），并缓存到 `_paths[(parentCoord, childCoord)]` 以便"走过的路径"染色。
- **存档时只存 children**，parents 由 `SavedActMap` 反向重建（`WireChildren`）。

### 1.5 生成规则（决定数据长什么样）

`StandardActMap` 流程：`GenerateMap()` → `AssignPointTypes()` → 剪枝 → 后处理。

- 宽度固定 **7 列**；每层只能向左/中/右移动 1 列（`col ±1`），禁止交叉线（`HasInvalidCrossover`）。
- 最后一层强制 `RestSite`；`RowCount - 7` 那层强制 `Treasure`（开启 `SwarmingElites` 时为 `Elite`）；第 1 层全部 `Monster`。
- 其余节点按配额随机，并满足约束：`row < 5` 不放 RestSite/Elite；`row >= 长度-3` 不放 RestSite；Elite/RestSite/Treasure/Shop **不能父子相邻、也不能兄弟相邻**。
- 后处理会**改坐标**：`MapPathPruning.PruneDuplicateSegments`（去重段）、`CenterGrid`（整体水平居中）、`SpreadAdjacentMapPoints`（同层拉开间距）、`StraightenPaths`（把"之"字形中间节点拉直）。
- 所以最终坐标不是"生成时"的坐标，而是后处理后的坐标。

### 1.6 绘制方式（仅理解坐标用，不需实现）

`NMapScreen.SetMap()`：`Position = new Vector2(col, row) * new Vector2(distX, -distY) + origin`，其中 `distY = 2325 / (rowCount - 1)`、`distX = 1050 / columnCount`，再叠加 ±21 / ±25 的抖动（用种子派生 RNG，保证同种子同布局）。图标由 `NNormalMapPoint.IconName(MapPointType)` 映射到贴图名（`map_monster` / `map_elite` / `map_shop` / `map_rest` / `map_chest` / `map_unknown`）。

### 1.7 结论：能用 ScriptableObject 吗

能，但必须**分三层**，别混在一个资产里：

| 数据层 | 例子 | 放哪 |
| --- | --- | --- |
| **静态配置** | 网格尺寸、节点列表、类型表现（图标/颜色）、类型配额与规则 | ✅ ScriptableObject（本文重点） |
| **生成结果 / 预设地图** | 一份固化的 coord + type + children 列表 | ✅ 也可以做成 SO，STS2 用 `SerializableActMap` JSON 存 |
| **运行时状态** | 已走 / 可走 / 不可走、当前所在节点 | ❌ 放运行时对象（`MonoBehaviour` / 普通 C# 类），**写进 SO 会污染资产** |

> ⚠️ Unity 序列化的两个硬约束（对应本文档的全部设计取舍）：
> 1. **不要用"节点对象引用节点对象"表示连线**。`List<MapNodeData>` 里的对象互指会让 Unity 序列化出错（循环引用 → null 或无限展开）。
> 2. **必须用坐标 `(col, row)` 当唯一 ID**，连线存 `List<Vector2Int> childCoords`。这正是 STS2 用 `List<MapCoord> ChildCoords` 的原因。

---

## 二、字段对照表

| STS2 | 本文 SO 模型 | 说明 |
| --- | --- | --- |
| `MapCoord.col` / `.row` | `MapNodeData.col` / `.row` | 层 = `row` |
| `MapPointType` | `MapPointType`（自定义 9 值） | 多了 WeakMonster / CorruptedMonster / CorruptedElite / Start |
| `MapPoint.Children` | `MapNodeData.childCoords: List<Vector2Int>` | 只存子节点 |
| `MapPoint.parents` | 不存，运行时反向构建 | 见 Step 6 |
| `MapPoint.CanBeModified` | `MapNodeData.locked`（可选） | 后处理时是否允许改坐标 |
| `ActMap.Grid[,]` | `MapConfig.columnCount` + `rowCount` + `nodes` | 由 nodes 反推，无需真的存二维数组 |
| `ActMap.startMapPoints` | `row == 0` 的节点 / 或 `isStart` 标记 | |
| `ActMap.BossMapPoint` | `MapPointType.Boss` 的节点 | |
| `MapPointTypeCounts` | `MapConfig` 上的配额字段（可选，Step 5） | 只有做随机生成才需要 |
| `MapPointState` | ❌ 不进 SO | 运行时状态 |

---

## 三、分步实现

建议目录：`Assets/Scripts/Map/`（SO 定义）+ `Assets/Data/Map/`（.asset 实例）。

### Step 1 — 定义节点类型枚举

文件：`Assets/Scripts/Map/MapPointType.cs`

```csharp
namespace Sts2Map
{
    /// <summary>地图节点类型。顺序即 Inspecter 下拉顺序。</summary>
    public enum MapPointType
    {
        Start = 0,            // 起点（对应 STS2 的 Ancient，row 0）
        Shop = 1,             // 商店
        Event = 2,            // 事件（STS2 的 Unknown，问号）
        WeakMonster = 3,      // 普通弱怪
        Monster = 4,          // 普通怪
        CorruptedMonster = 5, // 腐化普通怪
        Elite = 6,            // 精英怪
        CorruptedElite = 7,   // 腐化精英怪
        Boss = 8,             // Boss
    }
}
```

### Step 2 — 定义单个节点数据

文件：`Assets/Scripts/Map/MapNodeData.cs`

要点：`[Serializable]` + 无继承 + 只用 Unity 能序列化的类型（`int` / `enum` / `string` / `Vector2Int` / `List<Vector2Int>`）。

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

namespace Sts2Map
{
    [Serializable]
    public class MapNodeData
    {
        [Tooltip("列，从 0 开始")]
        public int col;

        [Tooltip("层，从 0 开始。row=0 为起点层")]
        public int row;

        public MapPointType type = MapPointType.Monster;

        [Tooltip("指向【下一层】子节点的坐标。连线由此产生：只存子节点，父节点运行时反推")]
        public List<Vector2Int> childCoords = new List<Vector2Int>();

        [Tooltip("后处理（居中/拉直/拉开间距）时是否允许移动该节点，对应 STS2 的 CanBeModified")]
        public bool locked;

        [Tooltip("备注，可放该节点固定的事件/遭遇 id")]
        public string note;

        public Vector2Int Coord { get { return new Vector2Int(col, row); } }

        public MapNodeData() { }

        public MapNodeData(int col, int row, MapPointType type)
        {
            this.col = col;
            this.row = row;
            this.type = type;
        }
    }
}
```

> `Vector2Int` 在 Inspector 里显示为 `X`(列) / `Y`(层)。若想显示成 `(col, row)` 的字段名，可自定义一个 `[Serializable] struct MapCoord { int col; int row; }` 替代，语义等价。

### Step 3 — 定义地图 ScriptableObject

文件：`Assets/Scripts/Map/MapConfig.cs`

只放**声明式数据 + 查询方法**，不放生成/绘制逻辑。

```csharp
using System.Collections.Generic;
using UnityEngine;

namespace Sts2Map
{
    [CreateAssetMenu(fileName = "MapConfig_Act01", menuName = "STS2/Map/Map Config", order = 0)]
    public class MapConfig : ScriptableObject
    {
        [Header("网格尺寸（仅作约束与校验用）")]
        [Min(1)] public int columnCount = 7;
        [Min(2)] public int rowCount = 15;

        [Header("地图标识")]
        [Tooltip("同一 Act 的多份地图配置用它区分，例如 act01_variant_a")]
        public string mapId = "act01";

        [Tooltip("随机生成时用的种子；=0 表示忽略")]
        public int seed;

        [Header("节点列表")]
        public List<MapNodeData> nodes = new List<MapNodeData>();

        // ---- 运行时索引（不参与序列化，需要时重建）----
        private Dictionary<Vector2Int, MapNodeData> _lookup;

        public void RebuildLookup()
        {
            if (_lookup == null) _lookup = new Dictionary<Vector2Int, MapNodeData>();
            _lookup.Clear();
            for (int i = 0; i < nodes.Count; i++)
            {
                if (nodes[i] == null) continue;
                _lookup[nodes[i].Coord] = nodes[i];
            }
        }

        public MapNodeData GetNode(int col, int row)
        {
            EnsureLookup();
            MapNodeData n;
            return _lookup.TryGetValue(new Vector2Int(col, row), out n) ? n : null;
        }

        public MapNodeData GetNode(Vector2Int coord)
        {
            return GetNode(coord.x, coord.y);
        }

        public bool HasNode(int col, int row)
        {
            return GetNode(col, row) != null;
        }

        /// <summary>取某一层的全部节点（层 = row）。</summary>
        public List<MapNodeData> GetNodesInRow(int row)
        {
            List<MapNodeData> result = new List<MapNodeData>();
            for (int i = 0; i < nodes.Count; i++)
            {
                if (nodes[i] != null && nodes[i].row == row) result.Add(nodes[i]);
            }
            return result;
        }

        /// <summary>取全部子节点（连线终点）。</summary>
        public List<MapNodeData> GetChildren(MapNodeData node)
        {
            List<MapNodeData> result = new List<MapNodeData>();
            if (node == null) return result;
            for (int i = 0; i < node.childCoords.Count; i++)
            {
                MapNodeData child = GetNode(node.childCoords[i]);
                if (child != null) result.Add(child);
            }
            return result;
        }

        /// <summary>取全部父节点。STS2 里 parents 是双向维护的，这里按需反推。</summary>
        public List<MapNodeData> GetParents(MapNodeData node)
        {
            List<MapNodeData> result = new List<MapNodeData>();
            if (node == null) return result;
            for (int i = 0; i < nodes.Count; i++)
            {
                MapNodeData n = nodes[i];
                if (n == null || n == node) continue;
                for (int j = 0; j < n.childCoords.Count; j++)
                {
                    if (n.childCoords[j] == node.Coord) { result.Add(n); break; }
                }
            }
            return result;
        }

        private void EnsureLookup()
        {
            if (_lookup == null || _lookup.Count != nodes.Count) RebuildLookup();
        }

        private void OnEnable()
        {
            RebuildLookup();
        }

#if UNITY_EDITOR
        private void OnValidate()
        {
            RebuildLookup();
        }
#endif
    }
}
```

> 注意 `EnsureLookup()` 用 `Count` 做粗略失效判断，够用；如果担心改用显式 `RebuildLookup()` 调用也可以。

### Step 4 — 编辑器校验（强烈建议）

继续写在 `MapConfig.cs` 里。规则对应第一部分 1.5 的生成约束，写错坐标时能立刻发现。

```csharp
        /// <summary>校验节点/连线合法性，返回是否通过；errors 里是全部问题。</summary>
        public bool Validate(out List<string> errors)
        {
            errors = new List<string>();
            Dictionary<Vector2Int, MapNodeData> map = new Dictionary<Vector2Int, MapNodeData>();

            // 1) 坐标唯一 + 不越界
            for (int i = 0; i < nodes.Count; i++)
            {
                MapNodeData n = nodes[i];
                if (n == null) { errors.Add("nodes[" + i + "] 为空引用"); continue; }

                Vector2Int c = n.Coord;
                if (map.ContainsKey(c)) errors.Add("坐标重复：(" + c.x + "," + c.y + ")");
                else map[c] = n;

                if (c.x < 0 || c.x >= columnCount) errors.Add("列越界：" + c + "（columnCount=" + columnCount + "）");
                if (c.y < 0 || c.y >= rowCount) errors.Add("层越界：" + c + "（rowCount=" + rowCount + "）");
            }

            // 2) 连线只能跨一层且横移不超过 1 列
            for (int i = 0; i < nodes.Count; i++)
            {
                MapNodeData n = nodes[i];
                if (n == null) continue;

                for (int j = 0; j < n.childCoords.Count; j++)
                {
                    Vector2Int cc = n.childCoords[j];
                    if (!map.ContainsKey(cc)) { errors.Add("节点 " + n.Coord + " 的子节点 " + cc + " 不存在"); continue; }
                    if (cc.y != n.row + 1) errors.Add("连线必须跨一层：" + n.Coord + " -> " + cc);
                    if (Mathf.Abs(cc.x - n.col) > 1) errors.Add("连线横移超过 1 列：" + n.Coord + " -> " + cc);
                }
            }

            // 3) 每层至少一个节点
            for (int r = 0; r < rowCount; r++)
            {
                bool any = false;
                for (int i = 0; i < nodes.Count; i++)
                {
                    if (nodes[i] != null && nodes[i].row == r) { any = true; break; }
                }
                if (!any) errors.Add("第 " + r + " 层没有任何节点");
            }

            // 4) 除最后一层外都要有子节点；除第一层外都要有父节点（可达性）
            for (int i = 0; i < nodes.Count; i++)
            {
                MapNodeData n = nodes[i];
                if (n == null) continue;

                if (n.row < rowCount - 1 && n.childCoords.Count == 0)
                    errors.Add("断头路：" + n.Coord + " 没有子节点");

                if (n.row > 0)
                {
                    bool hasParent = false;
                    for (int k = 0; k < nodes.Count && !hasParent; k++)
                    {
                        MapNodeData p = nodes[k];
                        if (p == null) continue;
                        for (int j = 0; j < p.childCoords.Count; j++)
                        {
                            if (p.childCoords[j] == n.Coord) { hasParent = true; break; }
                        }
                    }
                    if (!hasParent) errors.Add("不可达：" + n.Coord + " 没有父节点");
                }
            }

            // 5) Boss 数量
            int bossCount = 0;
            for (int i = 0; i < nodes.Count; i++)
            {
                if (nodes[i] != null && nodes[i].type == MapPointType.Boss) bossCount++;
            }
            if (bossCount == 0) errors.Add("缺少 Boss 节点");
            if (bossCount > 1) errors.Add("Boss 节点多于 1 个（共 " + bossCount + " 个）");

            return errors.Count == 0;
        }
```

配合一个编辑器按钮（可选，放 `Assets/Scripts/Map/Editor/MapConfigEditor.cs`）：

```csharp
#if UNITY_EDITOR
using System.Collections.Generic;
using UnityEditor;
using UnityEngine;

namespace Sts2Map.EditorTools
{
    [CustomEditor(typeof(MapConfig))]
    public class MapConfigEditor : Editor
    {
        public override void OnInspectorGUI()
        {
            DrawDefaultInspector();

            MapConfig cfg = (MapConfig)target;

            if (GUILayout.Button("校验配置"))
            {
                List<string> errors;
                if (cfg.Validate(out errors))
                {
                    Debug.Log("[MapConfig] 校验通过：" + cfg.nodes.Count + " 个节点");
                }
                else
                {
                    for (int i = 0; i < errors.Count; i++) Debug.LogError("[MapConfig] " + errors[i]);
                }
            }
        }
    }
}
#endif
```

### Step 5（可选）— 节点类型的表现配置

因为目标是用配置驱动绘制，建议把"图标 / 颜色"也抽成 SO，避免在代码里 `switch (type)` 硬编码。

文件：`Assets/Scripts/Map/MapNodeTypeCatalog.cs`

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

namespace Sts2Map
{
    [Serializable]
    public class MapNodeTypeVisual
    {
        public MapPointType type;
        public string displayName;
        public Sprite icon;
        public Color tint = Color.white;
    }

    [CreateAssetMenu(fileName = "MapNodeTypeCatalog", menuName = "STS2/Map/Node Type Catalog", order = 1)]
    public class MapNodeTypeCatalog : ScriptableObject
    {
        public List<MapNodeTypeVisual> visuals = new List<MapNodeTypeVisual>();

        public MapNodeTypeVisual Get(MapPointType type)
        {
            for (int i = 0; i < visuals.Count; i++)
            {
                if (visuals[i].type == type) return visuals[i];
            }
            return null;
        }
    }
}
```

> 只在 `MapConfig` 上挂一个 `public MapNodeTypeCatalog typeCatalog;` 引用即可，别把 visual 数据内联进每个节点。

### Step 6 — 运行时图对象（父节点反向构建）

SO 是只读资产，不要在运行时改它。需要遍历父子时，构建一层运行时对象（对应 STS2 的 `MapPoint` + `MapPointState`）：

文件：`Assets/Scripts/Map/MapNode.cs` / `MapGraph.cs`

```csharp
using System.Collections.Generic;
using UnityEngine;

namespace Sts2Map
{
    /// <summary>节点运行时状态（对应 STS2 MapPointState）。</summary>
    public enum MapNodeState { None, Travelable, Traveled, Untravelable }

    public class MapNode
    {
        public readonly MapNodeData Data;
        public readonly HashSet<MapNode> Parents = new HashSet<MapNode>();
        public readonly HashSet<MapNode> Children = new HashSet<MapNode>();

        public MapNodeState State = MapNodeState.None;

        public MapNode(MapNodeData data) { Data = data; }

        public MapPointType Type { get { return Data.type; } }
        public Vector2Int Coord { get { return Data.Coord; } }
        public int Row { get { return Data.row; } }
        public int Col { get { return Data.col; } }
    }

    /// <summary>由 MapConfig 构建的可遍历图，父子双向维护。</summary>
    public class MapGraph
    {
        private readonly Dictionary<Vector2Int, MapNode> _nodes = new Dictionary<Vector2Int, MapNode>();

        public int ColumnCount { get; private set; }
        public int RowCount { get; private set; }

        public MapGraph(MapConfig config)
        {
            ColumnCount = config.columnCount;
            RowCount = config.rowCount;

            for (int i = 0; i < config.nodes.Count; i++)
            {
                MapNodeData data = config.nodes[i];
                if (data == null) continue;
                _nodes[data.Coord] = new MapNode(data);
            }

            foreach (KeyValuePair<Vector2Int, MapNode> kv in _nodes)
            {
                List<Vector2Int> childCoords = kv.Value.Data.childCoords;
                for (int j = 0; j < childCoords.Count; j++)
                {
                    MapNode child;
                    if (!_nodes.TryGetValue(childCoords[j], out child)) continue;
                    kv.Value.Children.Add(child);
                    child.Parents.Add(kv.Value);   // 与 STS2 AddChildPoint 一致：双向维护
                }
            }
        }

        public MapNode GetNode(Vector2Int coord)
        {
            MapNode n;
            return _nodes.TryGetValue(coord, out n) ? n : null;
        }

        public ICollection<MapNode> AllNodes { get { return _nodes.Values; } }

        public List<MapNode> GetNodesInRow(int row)
        {
            List<MapNode> result = new List<MapNode>();
            foreach (KeyValuePair<Vector2Int, MapNode> kv in _nodes)
            {
                if (kv.Value.Row == row) result.Add(kv.Value);
            }
            return result;
        }
    }
}
```

### Step 7 — 怎么用

```csharp
public class MapLoader : MonoBehaviour
{
    public MapConfig config;               // Inspector 拖入 .asset

    private MapGraph _graph;

    private void Awake()
    {
        List<string> errors;
        if (!config.Validate(out errors))
        {
            for (int i = 0; i < errors.Count; i++) Debug.LogError(errors[i]);
        }

        _graph = new MapGraph(config);

        // 起点层可走
        List<MapNode> startRow = _graph.GetNodesInRow(0);
        for (int i = 0; i < startRow.Count; i++) startRow[i].State = MapNodeState.Travelable;
    }
}
```

在 Project 窗口右键 → `Create / STS2 / Map / Map Config` 生成 `.asset`，然后按 Step 8 的示例填数据。

### Step 8 — 数据示例

一份 `7 列 × 8 层`、含 8 种节点、可直接通过 Step 4 校验的最小数据（JSON 便于对照，Inspector 里逐条填入 `nodes`）：

```json
{
  "mapId": "act01",
  "columnCount": 7,
  "rowCount": 8,
  "nodes": [
    { "col": 3, "row": 0,  "type": "Start",  "childCoords": [ { "x": 3, "y": 1 }, { "x": 4, "y": 1 } ] },

    { "col": 3, "row": 1,  "type": "WeakMonster", "childCoords": [ { "x": 3, "y": 2 } ] },
    { "col": 4, "row": 1,  "type": "WeakMonster", "childCoords": [ { "x": 4, "y": 2 } ] },

    { "col": 3, "row": 2,  "type": "Event",   "childCoords": [ { "x": 2, "y": 3 } ] },
    { "col": 4, "row": 2,  "type": "Monster", "childCoords": [ { "x": 4, "y": 3 }, { "x": 5, "y": 3 } ] },

    { "col": 2, "row": 3,  "type": "Monster", "childCoords": [ { "x": 2, "y": 4 } ] },
    { "col": 4, "row": 3,  "type": "Shop",    "childCoords": [ { "x": 4, "y": 4 } ] },
    { "col": 5, "row": 3,  "type": "Elite",   "childCoords": [ { "x": 5, "y": 4 } ] },

    { "col": 2, "row": 4,  "type": "CorruptedMonster", "childCoords": [ { "x": 2, "y": 5 } ] },
    { "col": 4, "row": 4,  "type": "Monster",          "childCoords": [ { "x": 3, "y": 5 }, { "x": 4, "y": 5 } ] },
    { "col": 5, "row": 4,  "type": "Monster",          "childCoords": [ { "x": 5, "y": 5 } ] },

    { "col": 2, "row": 5,  "type": "Event",   "childCoords": [ { "x": 2, "y": 6 } ] },
    { "col": 3, "row": 5,  "type": "Shop",    "childCoords": [ { "x": 3, "y": 6 } ] },
    { "col": 4, "row": 5,  "type": "Monster", "childCoords": [ { "x": 4, "y": 6 } ] },
    { "col": 5, "row": 5,  "type": "CorruptedElite", "childCoords": [ { "x": 5, "y": 6 } ] },

    { "col": 2, "row": 6,  "type": "Monster", "childCoords": [ { "x": 3, "y": 7 } ] },
    { "col": 3, "row": 6,  "type": "Monster", "childCoords": [ { "x": 3, "y": 7 } ] },
    { "col": 4, "row": 6,  "type": "Elite",   "childCoords": [ { "x": 4, "y": 7 } ] },
    { "col": 5, "row": 6,  "type": "Monster", "childCoords": [ { "x": 4, "y": 7 } ] },

    { "col": 3, "row": 7,  "type": "Boss", "childCoords": [] }
  ]
}
```

> 这份数据共 20 个节点、8 层，最后一层收敛到 Boss。要按 STS2 做成 15 层，只需把 `rowCount` 改成 15，并在 `row 7` 与 `row 14` 之间补上中间层节点（Boss 移到 `row = rowCount - 1`）。层的数量由 `rowCount` 决定，`row` 就是每个节点的层号。

### Step 9 — 自检清单

- [ ] 每个节点 `(col,row)` 唯一，且在 `0 <= col < columnCount`、`0 <= row < rowCount` 内
- [ ] 所有 `childCoords` 都能在 `nodes` 里找到对应节点（**不要手写坐标写错**）
- [ ] 连线跨越恰好 1 层（`child.row == row + 1`）且 `|child.col - col| <= 1`
- [ ] 每层至少 1 个节点（`row` 从 0 到 `rowCount-1` 连续）
- [ ] 除最后一层外每个节点有子节点；除第 0 层外每个节点有父节点
- [ ] 恰好 1 个 `Boss`，且位置在最后一层
- [ ] `parents` **没有**写进 SO（只在 Step 6 的运行时对象里构建）
- [ ] 运行时状态（`MapNodeState`）**没有**写进 `MapConfig` 资产
- [ ] `MapConfig` 里没有 `List<MapNodeData>` 的自引用/交叉引用

### Step 10（可选）— 与 STS2 / RL 数据互通

若之后要和本仓库的 RL 链路（[bridge_mod/RlMapHandler.cs](file:///c:/Project/sts2-rl-agent/bridge_mod/RlMapHandler.cs)）或 STS2 存档对齐，按 `SerializableActMap` 的 JSON 形状导出即可：

```json
{
  "points": [ { "coord": { "col": 3, "row": 1 }, "type": 4, "children": [ { "col": 3, "row": 2 } ] } ],
  "boss":   { "coord": { "col": 3, "row": 7 }, "type": 8 },
  "start":  { "coord": { "col": 3, "row": 0 }, "type": 0 },
  "width": 7,
  "height": 8
}
```

对应源码：[SerializableActMap.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Saves.Runs/SerializableActMap.cs)、[SerializableMapPoint.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Saves.Runs/SerializableMapPoint.cs)。
注意 JSON 里用的是 `coord` 对象 + `children` 坐标数组——**和本文档的 `childCoords` 是同一个思路**。

---

## 四、下一步（超出本文范围）

1. **自动生成器**：按 `MapPointTypeCounts` 配额 + 1.5 的约束随机生成 `nodes`，替代手填。
2. **后处理**：居中 / 拉开间距 / 拉直，对应 `MapPostProcessing`。
3. **绘制**：读 `MapGraph` 的 `Children` 画连线，按 `MapNodeTypeCatalog` 取图标/颜色。
4. **运行时状态机**：`Traveled / Travelable` 的推进与存档。

---

## 五、Layer（层）的实现细节

补充说明"层"到底是什么，以及要不要为它单独建结构。

### 5.1 STS2 里没有叫 Layer 的类

层的本质就是 `MapCoord.row` 这一个 `int`，容器是 `ActMap.Grid` 的**第二维**：

```csharp
// ActMap.cs
protected abstract MapPoint?[,] Grid { get; }          // Grid[col, row]
public int GetColumnCount() { return Grid.GetLength(0); }  // 列数
public int GetRowCount()    { return Grid.GetLength(1); }  // ← 层数
public IEnumerable<MapPoint> GetPointsInRow(int row) { ... } // 取某一层的所有节点
```

所以：**层不是对象，是坐标的一个维度**。没有 `Layer` 类、没有 `List<Layer>`。

### 5.2 三个"层"的概念别混

| 名称 | 起点 | 含义 | 源码 |
| --- | --- | --- | --- |
| `MapCoord.row` | 0 | **地图层索引**（配置里的 `row`） | [MapCoord.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Map/MapCoord.cs#L8-L15) |
| `ActFloor` | 1 | **当前楼层**（HUD 显示的"第几层"） | [RunState.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Runs/RunState.cs#L90-L90) |
| `TotalFloor` | 1 | 跨 Act **累计**楼层 = 各 Act 的 `MapPointHistory` 条目数之和 | [RunState.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Runs/RunState.cs#L92-L92) |

### 5.3 层与楼层的换算（关键一行）

```csharp
// RunManager.cs
private Task EnterMapCoordInternal(MapCoord coord, AbstractRoom? preFinishedRoom, bool saveGame)
{
    MapPoint point = State.Map.GetPoint(coord);
    return EnterMapPointInternal(coord.row + 1, point.PointType, coord, ...);  // ← actFloor = row + 1
}
```

即 **`ActFloor = coord.row + 1`**：`row=0` 是起点（界面上第 1 层）。
地图总层数另有 `ActModel.GetNumberOfFloors() = GetNumberOfRooms(isMultiplayer) + 2`（[ActModel.cs](file:///c:/Project/sts2-rl-agent/decompiled/MegaCrit.Sts2.Core.Models/ActModel.cs#L185-L188)），多出的 2 层对应起点层与 Boss/第二 Boss 层。

### 5.4 在 SO 里怎么写

由 5.1 可知：**不需要为层建单独的数据结构**。`MapNodeData.row` 是层号，`MapConfig.rowCount` 是层数，取某层节点就是 `GetNodesInRow(row)`（见 Step 3）。层与层之间的连接由 `childCoords` 表达（必须指向 `row + 1`）。

### 5.5 可选：按层配置规则

如果要做 STS2 那种"低层不出精英/营火"的约束，就在 `MapConfig` 上加一个**按 row 索引**的规则列表——这是唯一值得为"层"单独建模的地方：

```csharp
    [Serializable]
    public class MapLayerRule
    {
        [Tooltip("层号，对应 MapNodeData.row")]
        public int row;

        [Header("该层允许出现的节点类型")]
        public bool allowShop = true;
        public bool allowEvent = true;
        public bool allowWeakMonster = true;
        public bool allowMonster = true;
        public bool allowCorruptedMonster = true;
        public bool allowElite = true;
        public bool allowCorruptedElite = true;

        [Header("强制类型（仅随机生成时生效）")]
        public bool forceType;
        public MapPointType forcedType = MapPointType.Monster;

        [Header("该层节点数量约束")]
        [Min(1)] public int minNodes = 1;
        [Min(1)] public int maxNodes = 7;
    }

    public List<MapLayerRule> layerRules = new List<MapLayerRule>();

    public MapLayerRule GetLayerRule(int row)
    {
        for (int i = 0; i < layerRules.Count; i++)
        {
            if (layerRules[i].row == row) return layerRules[i];
        }
        return null;
    }
```

对应的 STS2 原始规则（`StandardActMap`）：

- `_lowerMapPointRestrictions = { RestSite, Elite }`：`row < 5` 时该层禁用
- `_upperMapPointRestrictions = { RestSite }`：`row >= 地图长度 - 3` 时禁用
- `RowCount - 1` 层强制 `RestSite`；`RowCount - 7` 层强制 `Treasure`（开启精英腐化时为 `Elite`）
- `_parentMapPointRestrictions` / `_childMapPointRestrictions` / `_siblingPointTypeRestrictions`：Elite / RestSite / Treasure / Shop 不能**父子相邻**或**兄弟相邻**

前三条是"按层"的，适合放进 `MapLayerRule`；后一条是"按邻接关系"的，属于生成算法（第三节 Step 4 的 `Validate()` 可以顺手加一条检查）。

