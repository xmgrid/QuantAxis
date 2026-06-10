# Agent C · 策略引擎

> **依赖**：Agent A（StrategyDao, QuoteDao）, Agent B（IndicatorCalc）  
> **输出给**：Agent D, E  
> **工期**：2 周

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
