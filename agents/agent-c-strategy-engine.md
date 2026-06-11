# Agent C · 策略引擎 + Vibe 2.0 算法管线

> **依赖**：Agent A（StrategyDao, QuoteDao, VibeResultDao）, Agent B（IndicatorCalc + Vibe新增指标）  
> **输出给**：Agent D, E  
> **工期**：2.5 周（+0.5 周 Vibe 2.0 算法实现）

---

## 任务清单

### C1. 策略 DSL 定义与解析

**文件**：`lib/engine/strategy_parser.dart`

```dart
class StrategyParser {
  /// 输入 JSON 字符串 → 输出可执行的条件树
  static ConditionNode parse(String dslJson);

  /// 条件树 → JSON（用于编辑器实时预览）
  static String toJson(ConditionNode node);
}

// 条件节点
class ConditionNode {
  LogicOp logic;       // AND / OR
  List<Condition> conditions;
}

// 单个条件
class Condition {
  String factor;       // 'ma_position' / 'macd_cross' / 'pe_range' ...
  Map<String, dynamic> params;
}
```

**所有支持的因子**（参数化）：

```dart
const factorRegistry = {
  // === 技术面 ===
  'ma_position':   {'params': ['ma_list','relation'], 
     'desc': '均线排列', 'hint': 'MA5>MA10>MA20'},
  'ma_cross':      {'params': ['fast','slow','direction'], 
     'desc': '均线交叉', 'hint': 'MA5上穿MA20'},
  'macd_cross':    {'params': ['position'], 
     'desc': 'MACD交叉', 'hint': 'DIF上穿DEA+零轴上方'},
  'macd_divergence':{'params': ['type'], 
     'desc': 'MACD背离', 'hint': '顶背离/底背离'},
  'kdj_signal':    {'params': ['k_min','k_max','d_min','d_max'], 
     'desc': 'KDJ信号', 'hint': 'K<20超卖金叉'},
  'rsi_range':     {'params': ['min','max'], 
     'desc': 'RSI区间', 'hint': '30<RSI<70'},
  'vol_break':     {'params': ['n','m'], 
     'desc': '放量突破', 'hint': '量>m×n日均量'},
  'vol_shrink':    {'params': ['n','ratio'], 
     'desc': '缩量', 'hint': '量<ratio×n日均量'},
  'price_break':   {'params': ['n','direction'], 
     'desc': '价格突破', 'hint': '突破60日高点'},
  'boll_signal':   {'params': ['position'], 
     'desc': '布林带信号', 'hint': '突破上轨/下轨'},
  'adx_trend':     {'params': ['min_adx'], 
     'desc': 'ADX趋势', 'hint': 'ADX>25趋势明确'},
  'turnover_range':{'params': ['min','max'], 
     'desc': '换手率区间', 'hint': '3%<换手<7%活跃'},

  // === 基本面 ===
  'pe_range':      {'params': ['min','max'], 
     'desc': '市盈率PE', 'hint': '0<PE<25'},
  'pb_range':      {'params': ['min','max'], 
     'desc': '市净率PB', 'hint': 'PB<3'},
  'roe_min':       {'params': ['min'], 
     'desc': 'ROE最低', 'hint': 'ROE>15%'},
  'revenue_growth':{'params': ['min_growth'], 
     'desc': '营收增速', 'hint': '同比增长>20%'},
  'profit_growth': {'params': ['min_growth'], 
     'desc': '利润增速', 'hint': '同比增长>15%'},
  'dividend_yield':{'params': ['min_yield'], 
     'desc': '股息率', 'hint': '股息率>3%'},
  'debt_ratio':    {'params': ['max_ratio'], 
     'desc': '资产负债率', 'hint': '<60%'},
};
```

**DSL 示例**（均线多头 + 放量）：
```json
{
  "logic": "AND",
  "conditions": [
    {"factor": "ma_position", "params": {"ma_list": [5,10,20,60], "relation": "bullish_align"}},
    {"factor": "vol_break", "params": {"n": 20, "m": 2.0}}
  ]
}
```

**验收**：
- [ ] 所有 20+ 因子均可解析
- [ ] 嵌套 AND/OR 逻辑正确
- [ ] DSL 往返（parse → toJson → parse）结果一致
- [ ] 非法 JSON / 未知因子 / 缺失参数 → 返回明确错误

---

### C2. 策略执行引擎

**文件**：`lib/engine/strategy_runner.dart`

```dart
class StrategyRunner {
  final QuoteDao quoteDao;

  /// 对单只股票执行策略
  Future<({bool matched, double score, List<String> reasons})> 
    evaluateStock(Strategy strategy, String tsCode);

  /// 遍历股票池执行策略
  Future<List<ScanResult>> runStrategy({
    required Strategy strategy,
    required List<String> tsCodes,     // 股票池代码列表
    int? topN,                         // 仅返回前 N 名
    void Function(int done, int total)? onProgress,
  });
}
```

**执行流程**：
1. 解析 DSL → 条件树
2. 遍历 `tsCodes` → 对每只股票：
   a. 读取日线数据（从 QuoteDao）
   b. 计算所需技术指标（从 IndicatorCalc）
   c. 逐个条件评估
   d. 计算匹配分数（匹配条件数 / 总条件数 × 100 + 指标强度加成）
3. 按 score 降序排序
4. 可选取 Top N
5. 结果写入 `scan_results` 表缓存

**评分算法**：
```dart
double calculateScore(List<Condition> all, List<Condition> matched, IndicatorCalc calc) {
  double base = matched.length / all.length * 100;  // 匹配比例
  // 指标强度加成（±10 分）
  // - 均线排列离散度越小越好
  // - MACD 金叉角度越陡越好
  // - 量比越大越好（但有上限）
  double boost = ...;
  return (base + boost).clamp(0, 100);
}
```

**后台执行**：
- 股票数 > 50 时，自动放入 Isolate 执行
- 每批 20 只，更新进度回调
- 支持中途取消（`CancelToken`）

**验收**：
- [ ] 单策略 + 120 只股票 < 3 秒
- [ ] 评分与同花顺选股结果一致性 > 80%
- [ ] Isolate 计算不阻塞 UI
- [ ] 取消操作立即停止

---

### C3. 本地回测引擎

**文件**：`lib/engine/backtest_engine.dart`

```dart
class BacktestEngine {
  Future<BacktestResult> run({
    required Strategy strategy,
    required List<String> tsCodes,
    required DateTime startDate,
    required DateTime endDate,
    double initialCapital = 100000,
    double commission = 0.0003,       // 万三手续费
    double slippage = 0.001,           // 0.1% 滑点
  });
}

class BacktestResult {
  double totalReturn;         // 总收益率
  double annualReturn;        // 年化收益率
  double maxDrawdown;         // 最大回撤
  double sharpeRatio;         // 夏普比率
  double winRate;             // 胜率
  double profitLossRatio;     // 盈亏比
  int totalTrades;
  List<TradeRecord> trades;
  List<double> equityCurve;   // 每日权益
  List<double> benchmarkCurve;// 基准（沪深300）
}
```

**模拟规则**：
- 按日线收盘价成交
- 信号日次日以开盘价买入/卖出
- 考虑涨跌停限制（±10% 不可交易）
- 考虑手续费 + 滑点
- 等权重分配仓位

**验收**：
- [ ] 3 年回测 < 10 秒（100 只股票）
- [ ] 收益率曲线可绘制
- [ ] 最大回撤计算准确
- [ ] 基准对标沪深 300

---

### C4. 策略编辑器页面

**文件**：`lib/screens/strategy_editor_screen.dart`

页面结构（参考原型）：
1. 策略名称输入框
2. AND/OR 逻辑切换按钮组
3. 条件因子列表：
   - 每行：删除按钮 + 因子下拉选择 + 参数值显示
   - 「+ 添加条件因子」按钮
4. DSL JSON 实时预览
5. 「保存策略」按钮 → 写入 StrategyDao
6. 「立即扫描测试」按钮 → 调用 StrategyRunner.runStrategy

**验收**：
- [ ] 因子下拉列表包含全部 20+ 因子
- [ ] DSL 实时预览随编辑更新
- [ ] 保存后出现在「自定义」分类
- [ ] 扫描测试返回匹配数量

---

### C5. Vibe 2.0 内置算法引擎

**文件**：`lib/engine/vibe_engine.dart`

Vibe 2.0 算法管线的核心实现。所有参数从 `VibeConfig` 读取，引擎本身不硬编码任何阈值。

#### C5a. VibeConfig 参数模型

**文件**：`lib/models/vibe_config.dart`

```dart
class VibeConfig {
  final TrendConfig trend;
  final RotationConfig rotation;
  final FactorConfig factor;
  final FilterConfig filter;
  final GlobalConfig global;

  // 从 Strategy.configJson 反序列化
  factory VibeConfig.fromJson(Map<String, dynamic> json);
  Map<String, dynamic> toJson();

  // 工厂：内置默认参数
  factory VibeConfig.defaults();
}
```

各子配置类及参数表参见 `产品设计方案.html` §十「可配置参数体系」（共 30+ 参数，含默认值、范围、步长）。

#### C5b. VibeEngine 核心类

```dart
class VibeEngine {
  final QuoteDao quoteDao;
  final VibeConfig config;

  /// 运行完整管线：趋势 → 轮动 → 因子 → 交集 → 七条件
  /// 返回所有阶段的结果
  Future<VibePipelineResult> runPipeline({
    required List<String> tsCodes,
    void Function(VibePhase phase, int done, int total)? onProgress,
  });

  /// 阶段1：趋势策略
  Future<Set<String>> runTrend(List<DailyQuote> quotes);

  /// 阶段2：行业轮动（需全股票池数据）
  Future<Set<String>> runRotation(Map<String, List<DailyQuote>> allQuotes, Map<String, String> stockIndustries);

  /// 阶段3：多因子评分 + Top N 筛选
  Future<Set<String>> runFactor(Map<String, List<DailyQuote>> allQuotes);

  /// 求交集
  Set<String> computeIntersection(Set<String> trend, Set<String> rotation, Set<String> factor);

  /// 阶段4：七条件过滤器
  Future<List<String>> applySevenConditions(List<String> intersectionCodes, {required bool strictMode});
}
```

#### C5c. 算法伪代码

**趋势策略 (runTrend)**：
```
输入: 股票日线列表 quotes
前提: len(quotes) >= config.trend.minDataDays  (默认60)

if config.trend.requireCloseAboveMA:
    maFast = MA(quotes.close, config.trend.maFast)     // 默认20
    check: quotes[-1].close > maFast

if config.trend.requireMACross:
    maFast = MA(quotes.close, config.trend.maFast)
    maSlow = MA(quotes.close, config.trend.maSlow)     // 默认60
    check: maFast > maSlow

if config.trend.requireShortMomentum:
    check: quotes[-1].close > quotes[-config.trend.shortLookback].close  // 默认5日前

三个条件全部满足 → 纳入 trendSet
```

**行业轮动 (runRotation)**：
```
输入: allQuotes {code → quotes}, stockIndustries {code → industry}
for each 行业组:
    candidates = []
    for each code in 行业:
        ret = (close[-1] / close[-config.rotation.momentumPeriod] - 1) × 100  // 默认20日
        if ret > config.rotation.minReturn:  // 默认0%
            candidates.add((code, ret))
    topN = candidates按ret降序取前 config.rotation.maxPerIndustry  // 默认1
    topN中所有code → rotationSet
```

**多因子评分 (runFactor)**：
```
输入: allQuotes {code → quotes}
for each code:
    if len(quotes) < config.factor.minDataDays: continue  // 默认60日
    ret = (close[-1] / close[-config.factor.momentumPeriod] - 1) × 100  // 默认20日
    vol = std(quotes.pctChange)
    score = config.factor.baseScore + ret × config.factor.momentumWeight - vol × config.factor.volatilityPenalty
    // 默认: 50 + ret×1.5 - vol×2.0
    加入候选列表

按score降序排列，取前 config.factor.topN 名 → factorSet
```

**七条件过滤器 (applySevenConditions)**：
```
输入: 三策略交集股票代码列表, strictMode

for each code:
    quotes = 加载日线
    if len(quotes) < config.filter.minDataDays: skip      // 默认30日

    // ① 非ST
    if config.filter.c1ExcludeST && stockName含"ST": skip

    // ② N日涨幅             严格默认>5% / 宽松默认≥0%
    ret = (close[-1] / close[-(period+1)] - 1) × 100      // period默认10
    threshold = strictMode ? c2StrictMin : c2LooseMin
    if ret <= threshold: skip

    // ③ 量比               严格默认>1.5 / 宽松默认>1.0
    volRatio = todayVolume / mean(volume[-5:])
    threshold = strictMode ? c3StrictMin : c3LooseMin
    if volRatio <= threshold: skip

    // ④ 涨幅区间            严格3%-5% / 宽松0%-7%
    low = strictMode ? c4StrictLow : c4LooseLow
    high = strictMode ? c4StrictHigh : c4LooseHigh
    if pctChange < low || pctChange > high: skip

    // ⑤ 收盘 > MA(N)        默认MA5
    if close <= MA(close, c5MaPeriod): skip

    // ⑥ 资金流入             近3日成交额 > 前3日成交额
    if requireAmountData:
        amtRecent = sum(amount[-recentDays:])
        amtPrior = sum(amount[-(recentDays+priorDays):-recentDays])
        if amtRecent <= amtPrior: skip

    // ⑦ 今日超额成交         今日成交额 > N日均额
    if requireAmountData:
        amtAvg = mean(amount[-amtPeriod:])
        if todayAmount <= amtAvg: skip

    全部通过 → 加入结果列表
```

#### C5d. VibePipelineResult 数据结构

```dart
class VibePipelineResult {
  final Set<String> trendCodes;
  final Set<String> rotationCodes;
  final Set<String> factorCodes;
  final Set<String> intersectionCodes;         // 三策略交集
  final List<String> sevenFilterCodesLoose;     // 七条件宽松通过
  final List<String> sevenFilterCodesStrict;    // 七条件严格通过

  // 每只股票的详细评分数据
  final Map<String, VibeStockDetail> details;
}

class VibeStockDetail {
  final double ret20;          // 20日收益率
  final double volatility;     // 波动率
  final double factorScore;    // 多因子综合分
  final double volRatio;       // 量比
  final List<String> sevenConditionsPassed;  // 通过的七条件编号
}
```

#### C5e. 与现有 StrategyRunner 的关系

```
现有:  StrategyParser + StrategyRunner → 通用条件因子评估
新增:  VibeEngine → 内置算法管线（共享 QuoteDao + IndicatorCalc）

Vibe 2.0 种子策略 → VibeEngine 执行（性能优化：三策略共用一次数据遍历）
用户自定义策略 → StrategyRunner 执行（通用DSL评估）
```

**验收**：
- [ ] `VibeConfig.fromJson()` 正确解析完整配置JSON（含默认值回退）
- [ ] 趋势策略结果与 `docs/app.py` 参考实现一致性 > 95%
- [ ] 行业轮动结果与参考实现一致性 > 95%
- [ ] 多因子评分排序与参考实现一致性 > 95%
- [ ] 三策略交集结果正确（严格集合交运算）
- [ ] 七条件过滤器严格/宽松双模式均正确
- [ ] 200 只股票完整管线扫描 < 5 秒
- [ ] 所有参数从 VibeConfig 读取，引擎代码零硬编码阈值

---

### C6. 策略参数版本兼容

**文件**：`lib/engine/config_migration.dart`

```dart
class ConfigMigration {
  /// 将旧版策略配置升级到最新 schema
  static Map<String, dynamic> migrate(Map<String, dynamic> oldConfig, int fromVersion);
  
  /// 填充缺失字段为默认值（向前兼容）
  static Map<String, dynamic> fillDefaults(Map<String, dynamic> partial);
  
  /// 当前最新 schema 版本号
  static const int currentSchemaVersion = 2;
}
```

迁移规则：
- v1→v2: 新增 `trend`/`rotation`/`factor`/`filter`/`global` 五组参数，旧 `conditions_json` 保留为 fallback
- 任何缺失字段 → 自动填充 `VibeConfig.defaults()` 中的出厂默认值
- 不删除任何旧字段，仅追加新字段

**验收**：
- [ ] v1 策略配置（仅 conditions_json）可正常加载运行
- [ ] 缺失字段自动填充默认值，不会 crash
- [ ] 升级后保存 → schema_version 更新为当前版本
