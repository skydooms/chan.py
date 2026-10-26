# chan.py Code Wiki

本 Wiki 面向「想读懂 / 想改造 / 想复用」本仓库的开发者，按 **架构 → 数据管道 → 各元素模块 → 配置 → 数据接入 → 可视化 → 增量计算 → 应用与扩展** 的顺序组织。

> 说明：仓库根目录的 `README.md` 与 `quick_guide.md` 是项目作者面向使用者的原始文档（其中相当一部分内容描述的是**未开源的完整版**）。本 Wiki 的所有结论均以**本仓库真实源码**为准，并在文中直接标注源码位置，便于逐条核对。

---

## 1. 这套文档包含什么

| 文档 | 内容 | 适合谁 |
| --- | --- | --- |
| [01-架构总览](01-架构总览.md) | 分层架构、一次计算的完整调用链、对象关系、模块依赖、两种计算模式 | 所有人，先读这篇 |
| [02-K线与合并](02-K线与合并.md) | `CTime` / `CKLine_Unit` / `CCombine_Item` / `CKLine_Combiner` / `CKLine` / `CKLine_List` | 想理解数据底座与合并规则 |
| [03-笔-Bi](03-笔-Bi.md) | `CBi` 笔对象、`CBiList` 画笔状态机、虚笔、笔的成立条件、笔的 MACD 度量 | 想调笔参数或实现自己的笔算法 |
| [04-线段-Seg](04-线段-Seg.md) | 特征序列 `CEigen`/`CEigenFX`、`CSegListChan` 划段流程、`CSeg`、三种线段算法 | 想理解最复杂的划段逻辑 |
| [05-中枢-ZS](05-中枢-ZS.md) | 中枢区间构造、延伸、合并、三种中枢算法、背驰判断 | 想理解中枢与背驰 |
| [06-买卖点-BSPoint](06-买卖点-BSPoint.md) | 1/1p/2/2s/3a/3b 六类买卖点判定、存储结构、增量失效清理 | 想理解买卖点从哪来 |
| [07-配置体系](07-配置体系.md) | `CChanConfig` 全参数表、默认值、分级别精调后缀（`-buy`/`-sell`/`-seg`）、参数校验机制 | 想正确配置计算 |
| [08-数据接入与指标](08-数据接入与指标.md) | 4 种内置 `DataAPI` 实现、自定义数据源、`Math` 指标族、特征 `CFeatures` | 想接自己的行情源 / 加指标 |
| [09-可视化](09-可视化.md) | `PlotMeta` 坐标抽象、`CPlotDriver` 绘制流程与全部绘图方法、`CAnimateDriver` | 想画图或换绘图引擎 |
| [10-增量计算与策略开发](10-增量计算与策略开发.md) | `trigger_step`/`step_load`/`trigger_load`、多级别喂数与时间对齐、快照、`is_sure` 与信号消失、防未来函数 | 想写策略 / 做回测 / 实盘 |
| [11-应用示例与运行指南](11-应用示例与运行指南.md) | `main.py`、`Debug/strategy_demo*.py`、`App` GUI、安装运行步骤、常见报错 | 想跑起来 |
| [12-扩展开发与依赖关系](12-扩展开发与依赖关系.md) | 完整依赖矩阵、循环依赖的处理方式、五类扩展点、性能建议 | 想二次开发 |

---

## 2. 项目定位

`chan.py` 是一个**缠论（缠中说禅）技术分析计算框架**：输入一段（或多级别）K 线，输出结构化的缠论元素——合并 K 线、笔、线段、中枢、形态学买卖点，并支持画图与逐根 K 线的增量更新。

作者在根 `README.md` 中说明：本仓库是**开源裁剪版**，只包含静态计算能力（作者称公开版约 5300 行；含 GUI 与 demo，本仓库实际约 7200 行），不包含策略、交易引擎、机器学习特征/模型、离线数据更新等完整版模块（约 22000 行）。因此根 `README.md` 目录树里出现的 `Config/`、`CustomBuySellPoint/`、`ModelStrategy/`、`Trade/`、`OfflineData/` 等目录在本仓库中**并不存在**。

---

## 3. 核心术语（读源码前必须对齐）

| 术语 | 代码实体 | 含义 |
| --- | --- | --- |
| klu | `CKLine_Unit` | 单根原始 K 线（含 OHLC、时间、成交量等） |
| klc | `CKLine` | 经过包含关系处理后的「合并 K 线」，只有 high/low，没有 open/close |
| bi | `CBi` | 笔，由两个方向相反的合并 K 线（分型）连接而成 |
| seg | `CSeg` | 线段，由至少 3 笔构成，端点方向一致 |
| segseg | `CSegListComm`（`kl_list.segseg_list`） | 「线段的线段」，即把线段当笔再算一层 |
| zs | `CZS` | 中枢，至少两笔的重叠区间 |
| bsp | `CBS_Point` | 形态学买卖点（事后看一定成立） |
| cbsp | — | 自定义动力学买卖点，**本仓库未包含**（属于完整版的策略层） |
| is_sure | 各元素 `is_sure` | 元素是否已确定；`False` 表示尾部未确认元素，画图为虚线，后续可能变化 |

---

## 4. 仓库目录（本仓库真实内容）

```
.
├── Chan.py                 # 主类 CChan：多级别加载、时间对齐、计算调度、序列化
├── ChanConfig.py           # CChanConfig：全局配置入口，聚合各子模块配置
├── main.py                 # 最小可运行示例（BaoStock 日线 + matplotlib）
├── Common/                 # 枚举、时间、异常、缓存装饰器、工具函数
├── DataAPI/                # 数据源：BaoStock / AKShare / CCXT / CSV + 抽象基类
├── KLine/                  # CKLine_Unit、CKLine(合并K线)、CKLine_List、TradeInfo
├── Combiner/               # 通用合并器 CKLine_Combiner（K线/特征序列共用）+ 元素适配器
├── Bi/                     # CBi、CBiList、CBiConfig
├── Seg/                    # CSeg、特征序列 CEigen/CEigenFX、三种线段算法
├── ZS/                     # CZS、CZSList、CZSConfig
├── BuySellPoint/           # CBS_Point、CBSPointList、CBSPointConfig
├── Math/                   # MACD / BOLL / RSI / KDJ / Demark / 均线与极值 / 趋势线
├── ChanModel/Features.py   # 买卖点特征容器 CFeatures
├── Plot/                   # CPlotDriver(matplotlib)、CAnimateDriver、PlotMeta
├── Debug/                  # 6 个策略/机器学习 demo
├── App/                    # PyQt6 版「A股买点扫描器」GUI
└── Script/requirements.txt # 基础依赖
```

---

## 5. 十行代码上手

```python
from Chan import CChan
from ChanConfig import CChanConfig
from Common.CEnum import AUTYPE, DATA_SRC, KL_TYPE
from Plot.PlotDriver import CPlotDriver

chan = CChan(
    code="sz.000001",
    begin_time="2018-01-01",
    end_time=None,
    data_src=DATA_SRC.BAO_STOCK,     # 需要已安装 baostock
    lv_list=[KL_TYPE.K_DAY],         # 必须从大级别到小级别
    config=CChanConfig({"bi_strict": True}),
    autype=AUTYPE.QFQ,
)

print(len(chan[0].bi_list), len(chan[0].seg_list), len(chan[0].zs_list))
for bsp in chan.get_latest_bsp(number=5):
    print(bsp.klu.time, bsp.type2str(), "buy" if bsp.is_buy else "sell")

CPlotDriver(chan, plot_config={"plot_kline": True, "plot_bi": True,
                               "plot_seg": True, "plot_zs": True, "plot_bsp": True}).save2img("test.png")
```

- `chan[0]` / `chan[KL_TYPE.K_DAY]` 返回该级别的 `CKLine_List`，它是所有元素的取值入口。
- 单级别时可省去级别参数，见 [Chan.py](file:///workspace/Chan.py#L291-L297)。

---

## 6. 环境与运行

- **Python ≥ 3.11**（根 `README.md` 明确要求；本项目为计算密集型，3.11 相比 3.8 实测提速约 16%）。
- 基础依赖见 [requirements.txt](file:///workspace/Script/requirements.txt)：`baostock`、`matplotlib`、`numpy`、`pandas`、`requests`、`ipython`。
- 可选依赖（未写入 requirements，按需安装）：
  - `akshare`：`DATA_SRC.AKSHARE` 与 GUI
  - `ccxt`：`DATA_SRC.CCXT`
  - `PyQt6`：`App/ashare_bsp_scanner_gui.py`
  - `xgboost`：`Debug/strategy_demo5.py`、`strategy_demo6.py`
- 运行：`python3 main.py`（`main.py` 默认使用 BaoStock，需联网）。详见 [11-应用示例与运行指南](11-应用示例与运行指南.md)。

---

## 7. 文档中的约定

- 代码位置统一使用 `file:///workspace/...#L起-L止` 链接，行号对应当前提交。
- 「未开源」指根 `README.md` 有描述、但本仓库无对应源码的能力，Wiki 会显式标注，避免误导。
- 文中所有流程图使用 Mermaid，GitHub / 支持 Mermaid 的 Markdown 阅读器可直接渲染。
