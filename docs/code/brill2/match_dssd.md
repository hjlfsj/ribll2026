# MatchDssdEvent_V1 配对算法详解

!!! warning
    Situations don’t consider: two adjacent events to one adjacent event and more complex cases

# DSSD 探测器前端-后端条带配对算法

## 1. 总体流程

```
MatchDssdEvent(input, detector, output, h_energy_diff)
  │
  ├─ Reset(output)                        // 清空输出
  ├─ if (n_front == 0 || n_back == 0)     // 任一面无 hit，直接返回
  │     return
  │
  ├─ MatchSimple(input, ...)              // 快速路径：处理 1f-1b, 2f-1b, 1f-2b
  │     └─ 若匹配成功 → output.num > 0，跳过复杂匹配
  │
  ├─ if (output.num == 0)
  │     MatchComplex_v1(input, ...)       // 通用路径：处理任意 hit 数组合
  │
  └─ 按能量降序排列 output 中的粒子
```

### 调用时机

输入 `DssdEvent` 在进入 `MatchDssdEvent` 之前已通过 `ApplyDssdNormalize` 完成归一化，且每面的 hits 已按能量降序排列。

---

## 2. 数据结构

### 2.1 输入: DssdEvent

| 字段 | 类型 | 说明 |
|------|------|------|
| `front_num` | `int` | 前端 hit 数量 |
| `back_num` | `int` | 后端 hit 数量 |
| `front_strip[8]` | `int` | 前端条带号（按能量降序） |
| `back_strip[8]` | `int` | 后端条带号（按能量降序） |
| `front_energy[8]` | `double` | 前端能量（按能量降序） |
| `back_energy[8]` | `double` | 后端能量（按能量降序） |
| `front_time[8]` | `double` | 前端时间（按能量降序） |
| `back_time[8]` | `double` | 后端时间（按能量降序） |

### 2.2 输出: DssdMatchEvent

| 字段 | 类型 | 说明 |
|------|------|------|
| `num` | `int` | 配对出的粒子数（最多 8） |
| `front_strip[i]` | `double` | 粒子 i 的前端加权条带位置 |
| `back_strip[i]` | `double` | 粒子 i 的后端加权条带位置 |
| `energy[i]` | `double` | 粒子 i 的能量 |
| `time[i]` | `double` | 粒子 i 的时间 |
| `merge_tag[i]` | `int` | 配对类型（见第 5 节） |
| `energy_diff[i]` | `double` | 正背面能量差 |
| `x[i], y[i], z[i]` | `double` | 粒子 i 的物理坐标 |

### 2.3 内部结构: SideCombo

单面（front 或 back）的 hit 组合，在 `MatchComplex_v1` 内部使用。

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `int` | 0=单条, 1=相邻双条, 2=非相邻双条 |
| `strip1, strip2` | `int` | 涉及的条带号（双条时按能量降序） |
| `energy1, energy2` | `double` | 各条带能量（双条时按能量降序） |
| `total_energy` | `double` | 总能量 |
| `time1, time2` | `double` | 各条带时间 |
| `time` | `double` | 主要时间（取能量更大者的时间） |
| `idx1, idx2` | `int` | 在原始 hit 数组中的索引 |

### 2.4 内部结构: DeEntry

前端组合与后端组合的配对候选，按能量差排序。

| 字段 | 类型 | 说明 |
|------|------|------|
| `diff` | `double` | `|fe - be|` |
| `fi` | `int` | front_combos 中的索引 |
| `bi` | `int` | back_combos 中的索引 |

---

## 3. MatchSimple —— 快速路径

当 hit 数为 `(1,1)`、`(2,1)` 或 `(1,2)` 时走此路径。输入 hits 先按 **条带号升序** 重排。

### 3.1 1f-1b 匹配

直接比较 `|front_e[0] - back_e[0]| < match_tolerance`。

- 满足 → `merge_tag = 0`，输出 1 个粒子
- 不满足 → 返回 false，进入 MatchComplex_v1

### 3.2 2f-1b 匹配

尝试三种模式，取能量差最小的：

| 模式 | 含义 | 能量比较 |
|------|------|----------|
| 0 | 只用 front[0] | `|front_e[0] - back_e[0]|` |
| 1 | 只用 front[1] | `|front_e[1] - back_e[0]|` |
| 2 | 合并两个 front | `|(front_e[0]+front_e[1]) - back_e[0]|` |

若 `min_diff < match_tolerance`：

- 模式 0/1 → 单条匹配，`merge_tag = 0`
- 模式 2，front 相邻 → 相邻合并，`merge_tag = 1`，条带位置取加权平均 `WeightedStrip`
- 模式 2，front 不相邻 → 非相邻拆分，`merge_tag = 3`，输出 2 个粒子（各自用原始能量，不拆分）

### 3.3 1f-2b 匹配

与 3.2 对称，尝试三种模式，取能量差最小的：

| 模式 | 含义 | 能量比较 |
|------|------|----------|
| 0 | 只用 back[0] | `|back_e[0] - front_e[0]|` |
| 1 | 只用 back[1] | `|back_e[1] - front_e[0]|` |
| 2 | 合并两个 back | `|(back_e[0]+back_e[1]) - front_e[0]|` |

若 `min_diff < match_tolerance`：

- 模式 0/1 → 单条匹配，`merge_tag = 0`
- 模式 2，back 相邻 → 相邻合并，`merge_tag = 1`
- 模式 2，back 不相邻 → 非相邻拆分，`merge_tag = 3`，输出 2 个粒子（按比例拆分能量）

---

## 4. MatchComplex_v1 —— 通用路径

当 `MatchSimple` 未匹配成功（`output.num == 0`）时进入。处理任意 hit 数组合。

### 4.1 截断

每面最多取 8 个 hit（`n_front > 8 → n_front = 8`）。

### 4.2 构建组合 (build_combos)

为每面生成所有可能的 hit 组合：

- **单条组合**（type=0）：每个 hit 独立
- **双条组合**（type=1 或 2）：所有 C(n,2) 个两两配对，按条带号差是否为 1 区分相邻/非相邻

共 `n + C(n,2) = n(n+1)/2` 个组合。

每个组合记录：
- `type`: 0/1/2
- `strip1, strip2`: 条带号（按能量降序，即 i < j 时 energy[i] ≥ energy[j]）
- `energy1, energy2`: 各条带能量
- `total_energy`: 总能量
- `time`: 取能量更大者的时间

### 4.3 构建能量差候选列表 (de_entries)

遍历所有 front_combo × back_combo 组合，计算 `diff = |fe - be|`。

若 `diff < match_tolerance`，将该配对加入候选列表。

### 4.4 贪心匹配

将 de_entries 按 `diff` 升序排列，依次处理：

1. 跳过已使用的 hit 索引（`used_front_idx` / `used_back_idx`）
2. 能量条件检查：`fe > diff && be > diff`
   - 含义：正背面各自的总能量不能差太多。设 `fe ≥ be`，则等价于 `be > fe/2`，即小能量必须大于大能量的一半
3. 根据 `(fc.type, bc.type)` 确定 `match_type`（见第 5 节）
4. 调用 `AppendMatchResult` 填充输出
5. 标记使用的 hit 索引

当 `output.num ≥ 8` 时停止。

### 4.5 能量条件详解

`fe > diff && be > diff` 其中 `diff = |fe - be|`。

设 `fe ≥ be`，则 `diff = fe - be`：
- `fe > fe - be` → `be > 0`（正能量恒成立）
- `be > fe - be` → `2·be > fe` → **小能量必须大于大能量的一半**

即正面和背面的总能量差异不能超过较小能量本身。这在低能量事件中尤其重要，能有效过滤噪声引起的假配对。

---

## 5. 配对类型 (merge_tag)

| merge_tag | front 组合 | back 组合 | 含义 | 输出粒子数 |
|-----------|-----------|-----------|------|-----------|
| 0 | 单条 | 单条 | 简单单粒子 | 1 |
| 1 | 相邻双条 | 单条 | 相邻合并 vs 单条 | 1 |
| 1 | 单条 | 相邻双条 | 单条 vs 相邻合并 | 1 |
| 2 | 相邻双条 | 相邻双条 | 相邻合并 vs 相邻合并 | 1 |
| 3 | 非相邻双条 | 单条 | 非相邻 vs 单条（拆分） | 2 |
| 3 | 单条 | 非相邻双条 | 单条 vs 非相邻（拆分） | 2 |
| 4 | 非相邻双条 | 相邻双条 | 非相邻 vs 相邻（拆分） | 2 |
| 4 | 相邻双条 | 非相邻双条 | 相邻 vs 非相邻（拆分） | 2 |
| — | 非相邻双条 | 非相邻双条 | 不处理 | 0 |

### 各类型详细处理

#### type 0: 单条 vs 单条

```
能量: fe (front 单条能量)
时间: front 单条时间
条带: front 条带号, back 条带号
```

#### type 1: 相邻双条 vs 单条

```
能量: 相邻双条总能量
时间: 相邻双条中能量更大者的时间
条带: 加权平均条带号, 单条条带号
```

加权平均 `WeightedStrip(s1, e1, s2, e2)`：
- 若 `e1 * 0.1 > e2`（e2 可忽略）→ 返回 `s1`
- 否则 → 返回 `(s1 + s2) / 2.0`

#### type 2: 相邻双条 vs 相邻双条

```
能量: 相邻双条总能量（两对相同）
时间: 取四者中能量最大者的时间
条带: 两侧各取加权平均条带号
```

#### type 3: 非相邻双条 vs 单条

单条侧的能量按非相邻侧各条带能量比例拆分，输出 2 个粒子：

```
frac1 = e_nonadj1 / (e_nonadj1 + e_nonadj2)
frac2 = e_nonadj2 / (e_nonadj1 + e_nonadj2)

粒子 1: 能量 = e_single * frac1, 条带 = (nonadj.strip1, single.strip1)
粒子 2: 能量 = e_single * frac2, 条带 = (nonadj.strip2, single.strip1)
```

每个粒子的 `energy_diff` 为其拆分能量与对应非相邻条带能量的差的绝对值。

#### type 4: 非相邻双条 vs 相邻双条

与 type 3 类似，相邻侧的能量按非相邻侧各条带能量比例拆分，输出 2 个粒子。相邻侧条带位置取加权平均。

---

## 6. 辅助函数

### 6.1 WeightedStrip

```cpp
double WeightedStrip(int strip0, double energy0, int strip1, double energy1)
```

计算两个条带的加权平均位置。若 `energy1` 远小于 `energy0`（`energy1 < energy0 * 0.1`），则忽略之，返回 `strip0`；否则返回 `(strip0 + strip1) / 2.0`。

### 6.2 StripPosition

```cpp
double StripPosition(double center, double size, int strips, double strip)
```

将条带号转换为物理坐标（mm）：

```
position = center + size * ((strip + 0.5) / strips - 0.5)
```

### 6.3 FillPhysicalPosition

根据探测器配置计算粒子的 (x, y, z) 物理坐标。探测器 `t0d2` 有特殊的坐标映射（back 条带映射到 x，front 条带映射到 y 且翻转）。

### 6.4 AppendMatchResult

将匹配结果写入 `DssdMatchEvent`。若 `output.num ≥ 8` 则忽略。写入后自动调用 `FillPhysicalPosition` 计算物理坐标。

---

## 7. 输出后处理

`MatchDssdEvent` 在返回前将所有输出粒子按**能量降序**重新排列，确保下游代码始终看到能量最高的粒子在索引 0 位置。

---

## 8. 算法特点与限制

1. **贪心匹配**：按能量差升序贪心选取，不保证全局最优，但物理事件中 hit 冲突较少，实用性足够。
2. **非相邻对非相邻不处理**：`(fc.type=2, bc.type=2)` 的组合被跳过（需要 3+ 粒子同时命中）。
3. **最多 8 个输出粒子**：由 `DssdMatchEvent` 内部数组大小限制。
4. **输入必须已按能量降序**：`MatchComplex_v1` 依赖此前提构建组合和选取时间。
5. **能量条件**：`fe > diff && be > diff` 在低能量时有效过滤噪声假配对。