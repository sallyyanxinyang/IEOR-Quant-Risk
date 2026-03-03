# process_data.py 函数说明文档

## 目录

1. [总体流程](#总体流程)
2. [配置与常量](#配置与常量)
3. [文件发现与数据加载](#文件发现与数据加载)
   - [find_contracts()](#find_contracts)
   - [load_contract()](#load_contract)
   - [load_aiagent()](#load_aiagent)
4. [流动性分析](#流动性分析)
   - [find_liquid_start()](#find_liquid_start)
5. [合约展期与拼接](#合约展期与拼接)
   - [get_rollover_dates()](#get_rollover_dates)
   - [stitch()](#stitch)
6. [数据质量过滤](#数据质量过滤)
   - [filter_low_volume_days()](#filter_low_volume_days)
   - [filter_incomplete_days()](#filter_incomplete_days)
7. [主流程](#主流程)
   - [process_market()](#process_market)
   - [main 入口](#main-入口)

---

## 总体流程

整个脚本的处理逻辑可以用下面这张图概括：

```
原始合约 CSV 文件 (NQH20.csv, NQM20.csv, ...)
        │
        ▼
   find_contracts()        自动扫描文件夹，找到所有合约文件
        │
        ▼
   load_contract()         逐个读取，解析为标准 DataFrame
        │
        ▼
   find_liquid_start()     判断每个合约从哪天开始有足够的流动性
        │
        ▼
   get_rollover_dates()    确定相邻合约之间的展期（切换）日期
        │
        ▼
   stitch()                按展期日把多个合约拼成一条连续时间序列
        │
        ▼
   filter_low_volume_days()   去掉日总成交量极低的日子（节假日等）
        │
        ▼
   filter_incomplete_days()   去掉 1 分钟数据条数不足 90% 的日子
        │
        ▼
   导出 _clean.csv            最终可用于后续分析的干净数据
```

---

## 配置与常量

### `BASE_DIR`

```python
BASE_DIR = r'/Users/arinzhang/Desktop/Columbia Spring 2026 /IEOR 4703 MONTE CARLO SIMULATION/project data'
```

你本地存放所有期货数据的根目录。**这是整个脚本里唯一需要手动修改的地方。** 脚本会在这个目录下寻找各个市场对应的子文件夹。

### `OUT_DIR`

```python
OUT_DIR = os.path.join(BASE_DIR, '_cleaned_output')
```

清洗后的数据输出目录，会自动创建在 `BASE_DIR` 下面。处理完后你会在这里看到：

| 文件 | 内容 |
|------|------|
| `Nasdaq_NQ_clean.csv` | 清洗后的 Nasdaq 连续 1 分钟数据 |
| `AIAgent_Nasdaq_NQ.csv` | 处理后的 AIAgent 5 分钟快照 |
| `summary.csv` | 所有市场的处理汇总 |
| ... | 其他市场同理 |

### `MARKETS` 字典

每个市场的配置信息，包含四个字段：

| 字段 | 含义 | 示例 |
|------|------|------|
| `folder_name` | 你电脑上文件夹名 | `'Nasdaq'`、`'German Bunds - German Government Bonds'` |
| `contract_prefix` | 合约文件名前缀 | `'NQ'` → 会匹配 NQH20.csv, NQM20.csv 等 |
| `aiagent_file` | AIAgent 文件名 | `'AIAgent_Nasdaq.csv'` |
| `tick_size` | 该合约的最小变动价位（spread） | `0.25`（NQ）、`0.10`（GC）、`1.0`（VG） |

如果你将来要处理新市场，只需要在这个字典里加一条就行。

---

## 文件发现与数据加载

### `find_contracts()`

```python
def find_contracts(folder, prefix) -> list[str]
```

**作用：** 在指定文件夹中，自动查找所有以 `prefix` 开头的 `.csv` 文件，并排除 AIAgent 文件。

**参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `folder` | str | 文件夹路径，如 `.../project data/Nasdaq` |
| `prefix` | str | 合约前缀，如 `'NQ'` |

**返回值：** 排好序的文件路径列表，如 `['/.../NQH20.csv', '/.../NQM20.csv', '/.../NQU20.csv']`

**实现细节：**

- 使用 `glob.glob()` 做模式匹配 `NQ*.csv`
- 结果经过 `sorted()` 排序，保证按字母顺序（也就是按合约月份代码 F→G→H→... 排列）
- 过滤掉文件名包含 `'AIAgent'` 的文件，因为它们不是原始 OHLC 合约数据

**为什么需要自动发现：** 不同市场的合约数量不同（Nasdaq 有 3 个，HeatingOil 有 7 个），硬编码文件名不够灵活。自动发现意味着如果将来在文件夹里加了新合约，不用改代码。

---

### `load_contract()`

```python
def load_contract(filepath) -> pd.DataFrame
```

**作用：** 读取单个合约的原始 CSV 文件，将其解析为标准化的 DataFrame。

**参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `filepath` | str | CSV 文件的完整路径 |

**返回值：** 包含以下列的 DataFrame：

| 列名 | 类型 | 说明 |
|------|------|------|
| `datetime` | datetime64 | 精确到分钟的时间戳 |
| `Open` | float | 该分钟的开盘价 |
| `High` | float | 该分钟的最高价 |
| `Low` | float | 该分钟的最低价 |
| `Close` | float | 该分钟的收盘价 |
| `Volume` | int | 该分钟的成交量 |
| `date` | date | 日期（从 datetime 提取，用于后续按天分组） |

**输入数据格式示例：**

```
2019.03.15.15:54:00,7400.00000000,7400.00000000,7400.00000000,7400.00000000,1
```

**实现细节：**

- 原始 CSV 没有表头，所以用 `header=None` 读入，然后手动命名列
- 日期时间格式 `2019.03.15.15:54:00` 用 `pd.to_datetime(..., format='%Y.%m.%d.%H:%M:%S')` 解析
- 读入后按时间排序，确保数据是从早到晚排列的

---

### `load_aiagent()`

```python
def load_aiagent(filepath) -> pd.DataFrame
```

**作用：** 读取 AIAgent CSV 文件。这类文件和合约数据的格式完全不同，是 5 分钟间隔的价格快照。

**参数 / 返回值：** 类似 `load_contract()`，但输出列为 `datetime, price, volume, date`。

**输入数据格式示例：**

```
43832,0,0,8787.75,0
43832,0,5,8787.75,0
43832,1,0,8789,0
```

其中：
- `43832` 是 Excel 序列日期（Excel serial date），表示从 1899-12-30 起经过的天数
- `0` 和 `0` 分别是小时和分钟
- `8787.75` 是价格
- `0` 是成交量

**实现细节：**

- Excel 序列日期的转换：`datetime(1899, 12, 30) + timedelta(days=43832)` → `2020-01-02`
- 小时和分钟通过 `timedelta` 拼接到日期上，构成完整的 datetime

**注意：** AIAgent 数据很多行的 volume 都是 0，这是正常的——它本质上是价格快照，不一定每个时间点都有成交。

---

## 流动性分析

### `find_liquid_start()`

```python
def find_liquid_start(df) -> date
```

**作用：** 判断一个合约从哪一天开始有足够的流动性，可以被纳入分析。

**背景（来自 project PDF）：**

> "You should just use the data with high volume and high number of trades. The first few weeks or months may not be relevant at all."

期货合约从上市到到期，前期的成交量通常非常低（可能一天只有几笔交易），这些数据用来计算统计量是没有意义的。我们需要找到成交量开始"起来"的那个时间点。

**算法步骤：**

1. 按天聚合，算出每天的总成交量
2. 取日成交量的 **75th percentile**（即活跃期的正常水平）
3. 设阈值为 75th percentile × 10%（相当于找"成交量不再接近零"的起点）
4. 计算 **5 天滚动平均**（smoothing，避免偶尔一天有成交就误判）
5. 找到第一个滚动平均超过阈值的日期

**参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `df` | DataFrame | 某个合约的完整数据（从 `load_contract()` 返回） |

**返回值：** `datetime.date` — 流动性起始日期

**示例：** 对于 NQH20（Nasdaq 2020 年 3 月到期合约），原始数据从 2019-03-15 开始，但流动性起始日大约在 2019-12-13——前 9 个月的数据基本不可用。

---

## 合约展期与拼接

### `get_rollover_dates()`

```python
def get_rollover_dates(contracts) -> list[dict]
```

**作用：** 确定相邻合约之间的展期日——即应该在哪一天从旧合约切换到新合约。

**背景（来自 project PDF 图 1 的说明）：**

> "Before ESH20 expires, ESM20 becomes more liquid around March 17, 2020. For this data one would use ESH20 from around 12/15/2019 to around 3/17/2020 and ESM20 from around 3/18/2020 to around 6/18/2020."

在实际期货交易中，临近到期的旧合约（front month）成交量逐渐下降，下一个到期的新合约（back month）成交量逐渐上升。当新合约的日成交量超过旧合约时，就是切换的时间点。

**算法步骤：**

1. 对相邻两个合约，分别算出每天的总成交量
2. 取两者时间上重叠的日期（inner join on date）
3. 找到新合约日成交量连续 2 天超过旧合约的第一个日期
4. 如果没有连续 2 天的情况，就取第一次超过的日期
5. 如果完全没有超过（极端情况），就在旧合约最后一天切换

**参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `contracts` | list of (name, DataFrame) | 按时间排序的合约列表 |

**返回值：** 字典列表，每个字典包含：

```python
{'from': 'NQH20', 'to': 'NQM20', 'date': datetime.date(2020, 3, 16)}
```

**为什么要求连续 2 天：** 单独一天的成交量波动可能是随机的（比如某天有大单），要求连续 2 天超过能避免过早切换。

---

### `stitch()`

```python
def stitch(contracts, rollovers, liquid_starts) -> pd.DataFrame
```

**作用：** 根据流动性起始日和展期日，把多个合约拼接成一条连续的时间序列。

**逻辑：**

每个合约只取其"有效期"内的数据：

| 合约序号 | 起始日 | 结束日 |
|----------|--------|--------|
| 第 1 个 | `liquid_start` | 第 1 个展期日 |
| 第 2 个 | 第 1 个展期日 | 第 2 个展期日 |
| ... | ... | ... |
| 最后 1 个 | 上一个展期日 | 数据结束 |

**参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `contracts` | list of (name, DataFrame) | 所有合约 |
| `rollovers` | list of dict | `get_rollover_dates()` 的输出 |
| `liquid_starts` | dict | 每个合约的流动性起始日 `{name: date}` |

**返回值：** 拼接后的完整 DataFrame，额外增加 `contract` 列标记每行数据来自哪个合约。

**示例（Nasdaq）：**

```
NQH20: 2019-12-13 → 2020-03-16  (86126 rows, 80 days)
NQM20: 2020-03-16 → 2020-06-16  (87215 rows, 78 days)
NQU20: 2020-06-16 → 2020-09-18  (91920 rows, 81 days)
```

三段数据拼接为从 2019-12 到 2020-09 的连续时间序列。

**注意：** 起始日和结束日的边界是 `[start, end)`（左闭右开），避免同一天的数据出现在两个合约中。

---

## 数据质量过滤

### `filter_low_volume_days()`

```python
def filter_low_volume_days(df, min_daily_vol=100) -> pd.DataFrame
```

**作用：** 移除日总成交量极低的交易日。

**为什么需要这一步：** 拼接后的数据中可能包含一些节假日、半天交易日、或者市场异常清淡的日期。这些日子的成交量可能只有个位数或零，统计意义很小。

**参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `df` | DataFrame | — | 拼接后的数据 |
| `min_daily_vol` | int | 100 | 日总成交量低于此值的日期被丢弃 |

**返回值：** 过滤后的 DataFrame。

---

### `filter_incomplete_days()`

```python
def filter_incomplete_days(df, coverage=0.90) -> (pd.DataFrame, int, int, int)
```

**作用：** 移除 1 分钟数据条数（bars）不足的交易日。

**背景（来自 project PDF）：**

> "Days with many missing data/trades should be discarded (for example the number of trading minutes should be more than 90% to be considered)."

**算法步骤：**

1. 统计每天有多少条 1 分钟数据
2. 用 **75th percentile** 来估算"正常的一天应该有多少条数据"
   - 为什么用 75th 而不是 max？因为 max 可能是某个异常长的交易日，75th 更能代表典型水平
   - 例如 Nasdaq 的 75th percentile 约为 1366 分钟/天
3. 设阈值 = 估算值 × 90%
4. 低于阈值的日期全部丢弃

**参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `df` | DataFrame | — | 过滤了低成交量后的数据 |
| `coverage` | float | 0.90 | 最低覆盖率要求（project 要求 ≥ 90%） |

**返回值：** 四元组 `(df_clean, good_days, bad_days, expected_bars)`

| 返回值 | 说明 |
|--------|------|
| `df_clean` | 过滤后的 DataFrame |
| `good_days` | 保留的天数 |
| `bad_days` | 被丢弃的天数 |
| `expected_bars` | 估算的每天应有分钟数 |

---

## 主流程

### `process_market()`

```python
def process_market(name, cfg) -> dict | None
```

**作用：** 单个市场的完整处理流水线。把上面所有的 helper 函数串起来。

**执行顺序：**

```
1. find_contracts()           找到所有合约文件
2. load_contract() × N        逐个加载
3. find_liquid_start() × N    逐个判断流动性起始
4. get_rollover_dates()       算出展期日
5. stitch()                   拼接
6. filter_low_volume_days()   去掉低成交量日
7. filter_incomplete_days()   去掉不完整日
8. 导出 CSV
9. (可选) load_aiagent()      处理 AIAgent 文件
```

**参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `name` | str | 市场标识，如 `'Nasdaq_NQ'` |
| `cfg` | dict | 该市场的配置（来自 `MARKETS` 字典） |

**返回值：** 如果成功，返回一个汇总字典（包含行数、天数、日期范围等）；如果文件夹不存在或没数据，返回 `None`。

---

### main 入口

```python
if __name__ == '__main__':
```

**执行逻辑：**

1. 检查 `BASE_DIR` 是否存在，不存在就报错退出
2. 列出 `BASE_DIR` 下的所有子文件夹
3. 遍历 `MARKETS` 字典，对每个市场调用 `process_market()`
4. 汇总所有市场的处理结果，打印表格并导出 `summary.csv`

**典型输出：**

```
       market  tick_size  contracts   rows  days      start        end  avg_vol/min  avg_bars/day  days_dropped
    Nasdaq_NQ   0.250000          3 199261   146 2019-12-16 2020-09-17       387.03        1364.8            93
      Gold_GC   0.100000          4  77413    68 2024-04-01 2024-07-30        89.77        1138.4           175
     Bunds_RX   0.010000          3 157968   150 2025-03-03 2025-12-02       871.57        1053.1            88
       JPY_JY   0.000005          2  76304    56 2024-09-09 2024-12-12        95.29        1362.6           103
HeatingOil_HO   0.010000          7  59136    81 2022-02-16 2022-06-22        14.07         730.1           202
    GBPUSD_BP   0.000100          2 130487   100 2020-03-12 2020-09-09        66.15        1304.9           102
 EuroStoxx_VG   1.000000          2 120346   101 2021-12-14 2022-06-15       613.65        1191.5            54
```

---

## 输出数据格式

每个市场输出的 `_clean.csv` 包含以下列：

| 列名 | 类型 | 示例 | 说明 |
|------|------|------|------|
| `datetime` | string | `2019-12-16 00:00:00` | 精确到分钟的时间戳 |
| `Open` | float | `8546.75` | 该分钟的开盘价 |
| `High` | float | `8547.00` | 该分钟的最高价 |
| `Low` | float | `8546.75` | 该分钟的最低价 |
| `Close` | float | `8547.00` | 该分钟的收盘价 |
| `Volume` | int | `14` | 该分钟的成交量 |
| `contract` | string | `NQH20` | 数据来源合约 |
| `date` | string | `2019-12-16` | 日期 |

这份数据可以直接用于 project 后续步骤：计算 Range / RangeUp / RangeDn、EWMA 状态分类、经验概率密度函数等。
