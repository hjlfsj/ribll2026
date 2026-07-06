# MatchDssdEvent 配对算法详解

!!! warning
    Situations don’t consider: two adjacent events to one adjacent event and more complex cases

## 总体架构

```
MatchDssdEvent(input, detector, output, h_energy_diff)
│
├─ 1. Reset(output)                        // 清空输出
├─ 2. 任一面的 hit 数为 0 → 直接返回（无法配对）
├─ 3. MatchSimple(input, ...)              // 第一阶段：简单匹配
│      └─ 成功 → 跳过 MatchComplex
│      └─ 失败 → 进入 MatchComplex
├─ 4. MatchComplex(input, ...)             // 第二阶段：贪心迭代匹配
└─ 5. 按能量降序排序输出结果
```

核心设计思想：**先简单后复杂**。能用简单规则匹配的事件，不进入昂贵的全局搜索。

---

## 第一阶段：MatchSimple

只处理 **三种简单情况**，其余情况返回 `false`，交由 `MatchComplex` 处理。

### 情况 1：1f-1b（front 1 hit, back 1 hit）

```
if |E_front - E_back| < match_tolerance → 匹配成功
```

逻辑：两面各只有一个 hit，直接比较能量差。如果差在阈值内，就是同一粒子穿过两面。

### 情况 2：2f-1b（front 2 hits, back 1 hit）

**第一步：枚举三种可能的能量匹配方式，选能量差最小的**

| best_i | 含义 | 能量差 |
|--------|------|--------|
| 0 | front[0] 单独匹配 back[0] | `|E_f0 - E_b0|` |
| 1 | front[1] 单独匹配 back[0] | `|E_f1 - E_b0|` |
| 2 | front[0]+front[1] 合并匹配 back[0] | `|(E_f0+E_f1) - E_b0|` |

即在"两个 front hit 中选一个匹配"和"两个 front hit 合并后匹配"之间，选择能量差最小的方案。

**第二步：根据 best_i 输出结果**

- **best_i = 0 或 1**：选能量差最小的那个 front hit 作为匹配结果，另一个 front hit 丢弃（可能是噪声）。

- **best_i = 2**（两个 front hit 合并）：判断 front[0] 和 front[1] 是否相邻条带：
  - **相邻**（`|strip[0] - strip[1]| == 1`）：电荷共享效应。合并两个 hit 为一个 cluster，strip 位置用 `WeightedStrip` 计算加权平均，能量为两个 hit 之和，merge_tag = 2。
  - **不相邻**：两个独立的 front hit 各自与 back 配对，输出两条匹配结果。

### 情况 3：1f-2b（front 1 hit, back 2 hits）

与 2f-1b 完全对称，只是角色互换。

| best_i | 含义 |
|--------|------|
| 0 | back[0] 单独匹配 front[0] |
| 1 | back[1] 单独匹配 front[0] |
| 2 | back[0]+back[1] 合并匹配 front[0] |

- **best_i = 0 或 1**：选能量差最小的 back hit。
- **best_i = 2**：
  - **相邻**：合并为一个 cluster，merge_tag = 1。
  - **不相邻**：按能量比例拆分 front 的能量：`E_b0/(E_b0+E_b1)` 和 `E_b1/(E_b0+E_b1)` 的比例分配给两个 back hit，输出两条匹配结果。

---

## 第二阶段：MatchComplex

`MatchSimple` 返回 `false`（即 hit 数不是上述三种简单情况，或虽然 hit 数是 2f-1b/1f-2b 但能量差超过阈值）时，进入 `MatchComplex`。

### 总体思路：贪心逐 front hit 迭代

```
按 front hit 顺序逐个处理（i = 0, 1, ..., x_hit-1）
  对当前 front hit[i]，构建三种候选能量：
    mp_ex[0] = E_f[i]                    ← 单独使用
    mp_ex[1] = E_f[i] + E_f[i-1]        ← 与左边合并（仅当相邻且左边未被使用）
    mp_ex[2] = E_f[i] + E_f[i+1]        ← 与右边合并（仅当相邻）

  对每个候选能量，扫描所有剩余 back hit：
    尝试 1b 匹配：|候选能量 - E_b|
    尝试 2b 合并匹配：|候选能量 - (E_b + E_b_neighbor)|

  选择全局最优（能量差最小）的匹配方案
  输出匹配结果，从 back 池中移除已使用的 back hit
  继续下一个 front hit
```

### 候选能量构建（mp_ex）

```cpp
mp_ex[0] = x_e[i];                                                      // 单 hit
mp_ex[1] = x_e[i] + x_e[i-1];  // 仅当 |strip[i] - strip[i-1]| == 1 且 i-1 未被使用
mp_ex[2] = x_e[i] + x_e[i+1];  // 仅当 |strip[i] - strip[i+1]| == 1
```

`mp_ex[1]` = 999999.0 表示该候选无效（不相邻或已被占）。

### 扫描 back hit 池（mp_idy）

`mp_idy` 是一个 `map<int, int>`，存储 **剩余可用的 back hit**：`key = strip号, value = 原始索引`。每匹配成功就从 map 中删除。

对每个候选能量 `mp_ex[id_ex]`，遍历 `mp_idy` 中每个 back hit：

| 尝试的模式 | 能量差计算 | judeg_y |
|-----------|-----------|---------|
| **1b**：候选能量 vs 单个 back hit | `|mp_ex - E_b|` | 0 |
| **2b (pre)**：候选能量 vs `it + 前一个相邻 back hit` | `|mp_ex - E_b - E_prev|` | 1 |
| **2b (next)**：候选能量 vs `it + 后一个相邻 back hit` | `|mp_ex - E_b - E_next|` | 2 |

### 全局最优选择

在所有 `id_ex × back_hit × 3种模式` 中，选出能量差 **最小** 的那个组合，记下：
- `judeg_x`：0/1/2（front 是单独/左合并/右合并）
- `judeg_y`：0/1/2（back 是单独/与前合并/与后合并）
- `pid_y1`, `pid_y2`：涉及的 back strip 号

### 四种输出模式

| judeg_x | judeg_y | 含义 | merge_tag |
|---------|---------|------|-----------|
| 0 | 0 | 1f-1b | 0 |
| 0 | 1 或 2 | 1f-2b（back 合并） | 1 |
| 1 或 2 | 0 | 2f-1b（front 合并） | 2 |
| 1 或 2 | 1 或 2 | 2f-2b（两面都合并） | 3 |

每种模式都输出一条匹配结果，strip 位置用 `WeightedStrip` 计算加权平均，能量为两个 hit 之和。

### WeightedStrip 加权条带位置

```cpp
double WeightedStrip(int strip0, double energy0, int strip1, double energy1) {
    if (energy0 * 0.1 > energy1) return double(strip0);     // E0 >> E1，取 strip0
    return 0.5 * double(strip0 + strip1);                    // 否则取中点
}
```

当一条的能量远大于另一条（>10倍）时，位置取主导条带的位置；否则取中点。

---

## 第三阶段：按能量降序排序

```cpp
for (int i = 0; i < output.num - 1; ++i) {
    for (int j = i + 1; j < output.num; ++j) {
        if (output.energy[j] > output.energy[i]) {
            // 交换所有字段
        }
    }
}
```

简单的冒泡排序，按能量从大到小排列所有匹配结果。这确保了输出中高能事件排在前面。

---

## merge_tag 含义汇总

| merge_tag | 模式 | 说明 |
|-----------|------|------|
| 0 | 1f-1b | 标准单 hit 匹配 |
| 1 | 1f-2b | back 面两个 hit 合并为一个 cluster |
| 2 | 2f-1b | front 面两个 hit 合并为一个 cluster |
| 3 | 2f-2b | 两面各两个 hit，分别合并为 cluster |

---

## 直方图记录

每次匹配成功时，`h_energy_diff->Fill(energy_diff)` 记录该次匹配的能量差。`MatchSimple` 中 3 个分支各填充一次，`MatchComplex` 中填充一次。