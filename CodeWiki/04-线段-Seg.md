# 04 · 线段（Seg）

线段是项目里**最绕、最易错**的一层（作者在 quick_guide 中直言「没啥事不要尝试阅读」）。本篇把它拆成四块：线段对象 `CSeg`、线段列表骨架 `CSegListComm`、特征序列 `CEigen`/`CEigenFX`、以及基于特征序列的默认实现 `CSegListChan`。

源码：

- [Seg/Seg.py](file:///workspace/Seg/Seg.py#L14-L161)
- [Seg/SegListComm.py](file:///workspace/Seg/SegListComm.py#L15-L177)
- [Seg/SegListChan.py](file:///workspace/Seg/SegListChan.py#L9-L76)
- [Seg/Eigen.py](file:///workspace/Seg/Eigen.py#L8-L28)
- [Seg/EigenFX.py](file:///workspace/Seg/EigenFX.py#L12-L184)
- [Seg/SegConfig.py](file:///workspace/Seg/SegConfig.py#L5-L15)

---

## 1. 一句话理解线段

> 把「上升线段中的下降笔」（或「下降线段中的上升笔」）当成 K 线，对这些「K 线」做**包含合并 + 分型判定**，得到的顶/底分型就是线段的转折点。

这些被当成 K 线的笔，就是**特征序列**（`CEigen`）。所以：

- `CEigen` 复用 [CKLine_Combiner](file:///workspace/Combiner/KLine_Combiner.py#L13-L171)（K 线合并器），只是元素类型换成了 `CBi`；
- `CEigenFX` 负责凑齐「3 个特征序列元素」并判断是否构成本线段的结束分型；
- `CSegListChan` 负责驱动 `CEigenFX` 并落成 `CSeg` 对象。

---

## 2. 线段对象 `CSeg`

```python
CSeg(idx, start_bi, end_bi, is_sure=True, seg_dir=None, reason="normal")
```

### 2.1 构造约束

[Seg.py#L15-L44](file:///workspace/Seg/Seg.py#L15-L44)：

| 约束 | 说明 |
| --- | --- |
| `start_bi.idx == 0 or start_bi.dir == end_bi.dir or not is_sure` | 线段首尾笔方向必须一致；只有「未确定线段」允许例外 |
| `dir = end_bi.dir`（默认） | 线段方向取结束笔方向，也可由 `seg_dir` 强制指定 |
| `end_bi.idx - start_bi.idx < 2` → `is_sure = False` | 笔数不足 3 的段一定是虚段 |
| `check()` | 已确定线段：下降段起点必须高于终点、上升段起点必须低于终点；且 `end_bi.idx - start_bi.idx >= 2`（至少 3 笔） |

### 2.2 成员

| 成员 | 类型 | 说明 |
| --- | --- | --- |
| `idx` | int | 线段序号 |
| `dir` | `BI_DIR` | 方向 |
| `is_sure` / `used_to_be_sure` | bool | 是否确定 / 曾确定 |
| `start_bi` / `end_bi` | `CBi` 或 `CSeg` | 起止笔（在 `segseg_list` 里是子线段） |
| `bi_list` | List | 段内笔，**只能**通过 `update_bi_list` 维护 |
| `zs_lst` | `List[CZS]` | 段内中枢（由 `update_zs_in_seg` 填充，倒序加入） |
| `eigen_fx` | `CEigenFX` | 结束该线段的特征序列分型（绘图/调试用） |
| `seg_idx` | int | 所属「线段的分段」编号 |
| `parent_seg` | `CSeg` | 上层线段 |
| `bsp` | `CBS_Point` | 若线段尾是买卖点 |
| `support_trend_line` / `resistance_trend_line` | `CTrendLine` | 段内趋势线（笔数 ≥ 3 时构建） |
| `ele_inside_is_sure` | bool | 段内中枢是否已彻底固化（增量优化） |
| `reason` | str | 该段的生成原因（调试用，例如 `0seg_find_high`、`split_first_1st`） |

### 2.3 关键方法

| 方法 | 说明 |
| --- | --- |
| `_high()` / `_low()` | 上升段取起点笔 low / 终点笔 high；下降段取起点笔 high / 终点笔 low |
| `get_begin_val()` / `get_end_val()` | 直接代理首/尾笔的起止价格 |
| `get_begin_klu()` / `get_end_klu()` | 代理首/尾笔的端点 K 线 |
| `get_klu_cnt()` | 覆盖的原始 K 线根数 |
| `cal_klu_slope()` / `cal_amp()` / `cal_bi_cnt()` | 斜率 / 涨跌幅 / 笔数（特征计算常用） |
| `cal_macd_metric(algo, reverse)` | **只支持 `slope` 与 `amp`**，其他算法抛 `PARA_ERROR` |
| `update_bi_list(bi_lst, idx1, idx2)` | 批量挂入笔，设置 `bi.parent_seg`；≥3 笔时构建支撑/压力趋势线 |
| `add_zs(zs)` / `clear_zs_lst()` | 中枢挂接（头插，保持时间正序） |
| `get_first_multi_bi_zs()` | 段内第一个「非一笔中枢」 |
| `get_multi_bi_zs_lst()` / `get_multi_bi_zs_cnt()` | 所有非一笔中枢 / 其数量 |
| `get_final_multi_bi_zs()` | 段内最后一个「非一笔中枢」 |

> `cal_macd_metric` 只支持 `slope/amp` 是线段买卖点的硬限制：`CChanConfig` 里即便你配了 `macd_algo="peak"`，线段买卖点的配置也会被强制改成 `"slope"`（见 [ChanConfig.py#L125-L129](file:///workspace/ChanConfig.py#L125-L129)）。

---

## 3. 特征序列 `CEigen` 与 `CEigenFX`

### 3.1 `CEigen`：一笔 = 一根「K 线」

[Seg/Eigen.py#L8-L28](file:///workspace/Seg/Eigen.py#L8-L28)：

- 继承 `CKLine_Combiner[CBi]`，因此天然拥有 `try_add / test_combine / update_fx / get_peak_klu`；
- 额外维护 `gap`：当分型成立且「分型元素与前一元素之间有跳空」时置 `True`（[Eigen.update_fx](file:///workspace/Seg/Eigen.py#L13-L17)）；
- `GetPeakBiIdx()`：返回该特征序列元素中极值笔的 `idx` **再减 1**，即线段终点落在极值笔的前一笔（上升段取最高笔、下降段取最低笔，见 [Eigen.GetPeakBiIdx](file:///workspace/Seg/Eigen.py#L22-L28)）。

### 3.2 `CEigenFX`：三元素分型

[Seg/EigenFX.py](file:///workspace/Seg/EigenFX.py#L12-L184) 是线段判定的核心状态机。

```python
fx = CEigenFX(_dir: BI_DIR, exclude_included=True, lv=SEG_TYPE.BI)
fx.add(bi)   # 每喂一笔，返回是否形成分型
```

初始化：

- `dir`：**线段方向**（不是特征序列方向）；
- `kl_dir`：`UP` if 线段向上 else `DOWN`，作为第一个特征元素的合并方向种子；
- `ele[0..2]`：三个特征序列元素（`CEigen` 或 `None`）；
- `lst`：喂进来的所有笔；
- `exclude_included=True`：特征序列按「排除包含」模式处理（这是缠论原文的严格做法）。

`add(bi)` 的前提是 `bi.dir != self.dir`——**只有反向笔才属于本线段的特征序列**（上升线段只吃下降笔）。之后按顺序进入三个处理函数：

| 阶段 | 函数 | 行为 |
| --- | --- | --- |
| 第 1 元素 | `treat_first_ele` | 直接创建 `ele[0]` |
| 第 2 元素 | `treat_second_ele` | 先尝试并入 `ele[0]`；不能合并则创建 `ele[1]`；若 `ele[1]` 的高/低没有超越 `ele[0]`（上升段要求 `ele[1].high < ele[0].high`），说明前两元素不可能是分型 → `reset()` |
| 第 3 元素 | `treat_third_ele` | 先尝试并入 `ele[1]`（可带 `allow_top_equal`）；不能合并则创建 `ele[2]`；然后 `actual_break()` 检查是否真的突破；最后对 `ele[1]` 调用 `update_fx(ele[0], ele[2])`；若分型方向正确返回 True，否则 `reset()` |

### 3.3 「跳空」为什么要额外处理

`can_be_end()`（[#L82-L93](file:///workspace/Seg/EigenFX.py#L82-L93)）：

```python
if self.ele[1].gap:
    # 分型中间元素与前元素有缺口 → 需要后面出现「反向且合法」的特征序列分型来确认
    return self.find_revert_fx(bi_lst, end_bi_idx + 2, thred_value, break_thred)
else:
    return True if self.actual_break_flag else None
```

这与 quick_guide 的描述完全一致：

- **无跳空**：分型出现即确认线段结束（`is_sure=True`，画实线）；
- **有跳空**：必须再找到一个「反向的、合法的」特征序列分型才算确认；找不到则返回 `None`，表示「已到尾端仍未确认」→ 线段仍为虚段（`is_sure=False`）。

`find_revert_fx`（[#L159-L184](file:///workspace/Seg/EigenFX.py#L159-L184)）就是做这件事：以 `revert_bi_dir(first_bi_dir)` 构造一个反向 `CEigenFX`，从 `begin_idx` 开始**隔一根**喂入（`bi_list[begin_idx::2]`，因为只有同向笔属于特征序列），并对结果递归调用 `can_be_end`。

### 3.4 `actual_break`：防「假突破」

[#L117-L157](file:///workspace/Seg/EigenFX.py#L117-L157)：因为特征序列合并可能把突破「吃掉」，所以要额外验证第三元素是否真的越过了第二元素的极值：

- 直接判断 `ele[2].low < ele[1][-1]._low()`（上升段）等；
- 若第三元素只有一笔，还要看它后续第 2、3 笔是否继续突破；若都不满足，置 `actual_break_flag = False`，最终 `can_be_end` 返回 `None`（未确认）。

### 3.5 `reset()`：滑动/重建窗口

[#L67-L80](file:///workspace/Seg/EigenFX.py#L67-L80)：

- `exclude_included=True`（默认）：**清空并从 `lst[1:]` 重新喂**，因为排除包含后元素边界会变；
- `exclude_included=False`：窗口左移一格（`ele0←ele1, ele1←ele2`），并从第二元素起始 idx 之后继续。

### 3.6 `all_bi_is_sure()`

[#L105-L107](file:///workspace/Seg/EigenFX.py#L105-L107)：只有喂入的所有笔都 `is_used_to_be_sure`，且「最后证据笔」也是确定笔，才认为该分型可以产出**确定线段**。否则产出的线段 `is_sure=False`。

---

## 4. 线段列表骨架 `CSegListComm`

这是「模板方法」的基类：定义线段增删与左侧剩余笔处理的通用流程，具体划段算法由子类实现 `update()`。

### 4.1 数据结构与构造

```python
CSegListComm(seg_config=CSegConfig(), lv=SEG_TYPE.BI)
```

| 成员 | 说明 |
| --- | --- |
| `lst: List[CSeg]` | 线段序列 |
| `lv: SEG_TYPE` | `BI` 表示「笔→线段」，`SEG` 表示「线段→线段的分段」 |
| `config: CSegConfig` | 线段配置 |

### 4.2 新增线段 `try_add_new_seg` / `add_new_seg`

[#L131-L158](file:///workspace/Seg/SegListComm.py#L131-L158)：

1. **首段拆分**：若当前没有线段、`split_first_seg=True` 且 `end_bi_idx >= 3`，先在 `bi_lst[end_bi_idx-3::-1]` 中反向找极值笔；若该极值笔比第一笔的开头更极端（或就是第 0 笔），则先插入一个「首段」，再插入目标段。这样避免第一段方向与首尾价格矛盾。
2. 计算 `bi1_idx`（首笔）与 `bi2 = bi_lst[end_bi_idx]`，创建 `CSeg(...)` 并维护 `pre/next`；
3. `update_bi_list(bi_lst, bi1_idx, end_bi_idx)` 挂入段内笔。
4. `add_new_seg` 包了一层异常处理：首段若因「起止价格异常」创建失败，**静默返回 False**（让调用方继续推进），其他异常继续抛。

### 4.3 左侧剩余笔如何处理

这是 `left_seg_method` 配置生效的地方——当尾部剩余若干笔还不足以构成确定线段时：

| 函数 | 作用 |
| --- | --- |
| `collect_first_seg` | 还没有任何线段时的首段处理。`peak` 模式：比较「最高-起点」与「起点-最低」的幅度，往更大的一侧先成虚段，再递归处理剩余；`all` 模式：直接用最后一笔成一段 |
| `collect_left_seg` | 左侧剩余笔处理的调度：无线段 → `collect_first_seg`；否则 `collect_segs`，若最后一段仍不确定且落后超过 2 笔则丢弃最后一段重算 |
| `collect_segs` | 尾部若出现「突破最后确定段尾部」的笔，则强制找极值笔成虚段；否则按 `left_method` 分派 |
| `collect_left_seg_peak_method` | `peak` 模式递归：从 `last_seg_end_bi.idx+3` 起找极值笔，找到就成虚段并继续，找不到就把剩余笔整体成段 |
| `collect_left_as_seg` | 把剩余笔整体收成一段（方向取决于首尾笔方向是否一致） |

> `left_seg_method="peak"`（默认）会尽量在「靠谱的新极值」处再切一刀；`"all"` 则把剩余笔一把收进一段。quick_guide 提示：`all` 容易找不到二类买卖点。

### 4.4 `FindPeakBi`

[#L168-L177](file:///workspace/Seg/SegListComm.py#L168-L177)：在笔序列中找极值笔，并额外排除「前两笔的终点更极端」的情况（避免找到已经被更早极值否定的点）。返回的是**笔**，不是价格。

---

## 5. 默认线段算法 `CSegListChan`

源码：[Seg/SegListChan.py](file:///workspace/Seg/SegListChan.py#L9-L76)

### 5.1 `do_init()`：回退未确定部分

[#L13-L26](file:///workspace/Seg/SegListChan.py#L13-L26)：

1. 从尾部弹掉所有 `is_sure=False` 的线段，并清空其内部笔的 `parent_seg`、断开 `next`；
2. 若最后一条**确定**线段的分型第三元素中，最后一笔仍不确定，也要把这条线断掉重算——因为「分型元素的高低点可能因虚笔变化而不准」。

### 5.2 `update()`：两段式

```python
def update(self, bi_lst):
    self.do_init()
    self.cal_seg_sure(bi_lst, begin_idx=0 if 空 else 最后一段 end_bi.idx+1)
    self.collect_left_seg(bi_lst)
```

- 先算**确定线段**（从最后一个确定位置往后）；
- 剩余笔交给基类的 `collect_left_seg` 生成虚段。

### 5.3 `cal_seg_sure()`：驱动两个方向的分型

[#L36-L62](file:///workspace/Seg/SegListChan.py#L36-L62)：

```python
up_eigen = CEigenFX(BI_DIR.UP)      # 上升线段的特征序列：下降笔
down_eigen = CEigenFX(BI_DIR.DOWN)  # 下降线段的特征序列：上升笔
for bi in bi_lst[begin_idx:]:
    if bi.is_down() and last_seg_dir != UP:   up_eigen.add(bi)
    elif bi.is_up() and last_seg_dir != DOWN: down_eigen.add(bi)
    if 形成分型: treat_fx_eigen(...); break
```

几个细节：

- **第一段方向不能「谁先成分型就用谁」**，否则会出现 US.EVRG 那类反例。源码用 `last_seg_dir` 与 `ele[1]` 是否存在做双向试探，并允许回退重试（[#L48-L58](file:///workspace/Seg/SegListChan.py#L48-L58)）；
- 一旦某方向形成分型，就 `treat_fx_eigen` 并**立即 break**，把剩余笔留给下一次 `cal_seg_sure` 递归处理；
- `last_seg_dir` 防止连续同向段。

### 5.4 `treat_fx_eigen()`：落成线段

[#L64-L76](file:///workspace/Seg/SegListChan.py#L64-L76)：

```python
_test = fx_eigen.can_be_end(bi_lst)      # True / None / False
end_bi_idx = fx_eigen.GetPeakBiIdx()     # 线段终点笔
if _test in (True, None):
    is_true = (_test is not None)        # None = 找到尾部仍未确认
    if not self.add_new_seg(bi_lst, end_bi_idx,
                            is_sure=is_true and fx_eigen.all_bi_is_sure()):
        # 首段方向与首尾价格冲突 → 跳过这笔重试
        self.cal_seg_sure(bi_lst, end_bi_idx + 1); return
    self.lst[-1].eigen_fx = fx_eigen
    if is_true:
        self.cal_seg_sure(bi_lst, end_bi_idx + 1)
else:
    # 分型不成立 → 从分型第二元素之后重新找
    self.cal_seg_sure(bi_lst, fx_eigen.lst[1].idx)
```

`is_sure` 的取值逻辑值得记住：**「能结束」且「所有笔曾确定」才是实段**；只有「能结束」但笔不确定，或「到尾部也没确认」，都只能产出虚段。

---

## 6. 其他两种线段算法（已废弃但保留）

| 算法 | `seg_algo` | 实现 | 思路 | 状态 |
| --- | --- | --- | --- | --- |
| 特征序列（默认） | `chan` | `CSegListChan` | 本文第 3~5 节 | ✅ 维护中 |
| 都业华 1+1 终结 | `1+1` | `CSegListDYH` | 用 `situation1/situation2` 判断笔的破坏关系来终结线段 | ⚠️ 代码内打印 deprecated 提示 |
| 线段破坏定义 | `break` | `CSegListDef` | 用「是否突破前一同向笔的极值」判定线段延伸/结束 | ⚠️ 代码内打印 deprecated 提示 |

选择逻辑在 [get_seglist_instance](file:///workspace/KLine/KLine_List.py#L19-L32)，未知取值直接抛 `PARA_ERROR`。

---

## 7. 线段配置 `CSegConfig`

[Seg/SegConfig.py](file:///workspace/Seg/SegConfig.py#L5-L15)

| 参数 | 取值 | 默认 | 说明 |
| --- | --- | --- | --- |
| `seg_algo` | `chan` / `1+1` / `break` | `chan` | 线段算法 |
| `left_seg_method` | `peak` / `all` | `peak` | 尾部剩余笔如何成段 |

---

## 8. 调试建议

- 线段画得「不对」时，先打开 `plot_eigen`（特征序列）和 `plot_segeigen`，肉眼确认 3 个特征元素与分型位置（[09-可视化](09-可视化.md)）；
- `CSeg.reason` 字段记录了每段的生成原因，出问题时打印 `seg.idx, seg.reason, seg.is_sure` 往往能快速定位是哪条分支导致的；
- 若所有线段都是虚段（`is_sure=False`），说明数据量不足或特征序列分型尚未确认，此时不应据此交易（quick_guide 明确提醒）。

---

## 9. 本篇小结

- 线段 = 对「反向笔构成的特征序列」做合并 + 分型；
- `CEigen` 复用 K 线合并器，`CEigenFX` 管理三元素分型与跳空确认，`CSegListChan` 驱动它们并落成 `CSeg`；
- 跳空分型必须等反向合法分型确认，这是「虚段」大量存在的原因；
- 尾部剩余笔由 `left_seg_method` 决定怎么收，直接影响中枢和 2 类买卖点。

下一步：[05-中枢-ZS](05-中枢-ZS.md)。
