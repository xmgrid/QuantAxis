# Agent D · 选股扫描 + 种子策略

> **依赖**：Agent A (StrategyDao), Agent B (KlinePainter), Agent C (StrategyRunner)  
> **输出给**：Agent G  
> **工期**：1.5 周

---

## 任务清单

### D1. 选股策略列表页

**文件**：`lib/screens/screener_screen.dart`

页面结构（参考原型）：
1. **策略分类标签栏**：
   - 全部(11) / 技术面(5) / 基本面(4) / 自定义(N)
   - 点击标签过滤策略列表
   - 标签下方显示当前筛选的策略数量

2. **行业筛选芯片行**：
   - 全部 / 半导体 / 食品饮料 / 电力设备 / 医药生物 / 银行 / 汽车 / 计算机 / 有色金属
   - 点击芯片高亮，联动扫描按钮文案

3. **种子策略高亮卡片**（金色边框）：
   - 标题：`[种子策略] 三大算法策略`
   - 描述：趋势策略 · 行业轮动 · 多因子
   - 点击进入种子策略详情页

4. **扫描按钮**：
   - `▶ 开始扫描选股 (11 策略 × 全行业)`
   - 文案随策略筛选和行业筛选动态更新

5. **策略卡片列表**：
   - 每个策略一张卡片，显示：名称 / 描述 / 分类·标签 / 匹配数
   - 预置策略有左侧金色竖条（`running` 样式）
   - 自定义策略有 ⚙️ 标记
   - 点击 → 进入策略结果页

6. **「＋ 创建自定义策略」卡片**（虚线边框）
   - 点击 → 进入策略编辑器（Agent C4）

```dart
class ScreenerScreen extends StatefulWidget {
  // 使用 Provider 获取 StrategyDao
}

class _ScreenerScreenState extends State<ScreenerScreen> {
  String _strategyFilter = 'all';      // all / technical / fundamental / custom
  String _industryFilter = 'all';

  List<Strategy> get _filteredStrategies {
    var list = _strategyFilter == 'all' 
      ? allStrategies 
      : allStrategies.where((s) => s.category == _strategyFilter).toList();
    return list;
  }
}
```

**验收**：
- [ ] 策略分类切换流畅（< 200ms）
- [ ] 行业芯片点击高亮
- [ ] 扫描按钮文案动态更新
- [ ] 自定义策略数量 > 0 时「自定义(N)」正常显示

---

### D2. 种子策略详情页

**文件**：`lib/screens/seed_strategies_screen.dart`

展示三大算法策略（参考原型）：

#### 种子一 · 趋势策略
- 算法原理说明
- 因子标签：MA5/10/20/60排列 · ADX>25 · 价格>MA60 · 布林带开口 · 新高突破
- 匹配数量 + 「▶ 查看趋势策略结果」按钮

#### 种子二 · 行业轮动策略
- 算法原理说明
- 因子标签：行业RPS排名 · 板块资金流向 · 行业内龙头筛选 · 月度调仓
- 当前强势行业展示
- 匹配数量 + 「▶ 查看轮动策略结果」按钮

#### 种子三 · 多因子策略
- 算法原理说明
- 因子标签：估值因子 · 质量因子 · 成长因子 · 动量因子 · 波动因子
- 综合排名说明
- 匹配数量 + 「▶ 查看多因子策略结果」按钮

```dart
class SeedStrategyCard extends StatelessWidget {
  final String title;
  final String subtitle;
  final String algorithmDesc;
  final List<String> factorTags;
  final int matchCount;
  final Color accentColor;
  final VoidCallback onViewResults;
}
```

**每个种子策略的执行**：
```dart
// 趋势策略
final trendStrategy = StrategyParser.parse('''
{
  "logic": "AND",
  "conditions": [
    {"factor": "ma_position", "params": {"ma_list": [5,10,20,60], "relation": "bullish_align"}},
    {"factor": "adx_trend", "params": {"min_adx": 25}},
    {"factor": "price_break", "params": {"n": 60, "direction": "high"}},
    {"factor": "boll_signal", "params": {"position": "upper_break"}}
  ]
}
''');

// 行业轮动策略
final rotationStrategy = StrategyParser.parse('''
{
  "logic": "AND",
  "conditions": [
    {"factor": "industry_rps", "params": {"top_n": 3}},
    {"factor": "vol_break", "params": {"n": 20, "m": 1.5}},
    {"factor": "price_break", "params": {"n": 20, "direction": "high"}}
  ]
}
''');

// 多因子策略
final multiFactorStrategy = StrategyParser.parse('''
{
  "logic": "AND",
  "conditions": [
    {"factor": "pe_range", "params": {"min": 0, "max": 50}},
    {"factor": "roe_min", "params": {"min": 10}},
    {"factor": "revenue_growth", "params": {"min_growth": 10}},
    {"factor": "profit_growth", "params": {"min_growth": 10}}
  ]
}
''');
```

**验收**：
- [ ] 三个种子策略全部可查看
- [ ] 每个策略的结果页显示匹配股票列表+排名+评分
- [ ] 点击股票跳转个股详情

---

### D3. 扫描页面（Top N + 进度动画）

**文件**：`lib/screens/scan_screen.dart`

页面结构（参考原型）：
1. Top N 选择器：Top 5 / Top 10 / Top 20 / Top 50 / 全部
2. 扫描进度区：
   - 旋转动画 spinner
   - 进度条（百分比）
   - 实时计数：已扫描 X/120 · 匹配 N · 耗时 X.Xs
3. 结果列表：
   - 排名序号（前 3 名金色）
   - 评分 badge（金色圆角标签）
   - 股票名称/代码/行业
   - 价格 + 涨跌幅

```dart
class ScanScreen extends StatefulWidget {
  final List<Strategy> strategies;
  final String industryFilter;
}

class _ScanScreenState extends State<ScanScreen> {
  int _topN = 10;
  double _progress = 0;
  int _scanned = 0;
  int _matched = 0;
  List<ScanResult> _results = [];
  bool _isScanning = false;
  CancelToken? _cancelToken;

  Future<void> _startScan() async {
    setState(() { _isScanning = true; _progress = 0; });
    final runner = StrategyRunner();
    _results = await runner.runStrategy(
      strategy: widget.strategies.first,
      tsCodes: _getFilteredStocks(),
      topN: _topN,
      onProgress: (done, total) {
        setState(() {
          _scanned = done;
          _progress = done / total;
        });
      },
    );
    setState(() { _isScanning = false; });
  }
}
```

**验收**：
- [ ] 进度动画流畅（200ms 更新间隔）
- [ ] Top N 切换即时重排结果
- [ ] 扫描完成自动展示结果
- [ ] 取消扫描无残留

---

### D4. 通用策略结果列表组件

**文件**：`lib/widgets/ranked_stock_list.dart`

```dart
class RankedStockList extends StatelessWidget {
  final List<RankedStock> stocks;
  // 每行：排名序号 + 评分 + 股票信息 + 价格涨跌
}

class RankedStock {
  final int rank;
  final double score;
  final String tsCode;
  final String name;
  final String industry;
  final double price;
  final double changePct;
}
```

**验收**：
- [ ] 种子策略结果页使用此组件
- [ ] 扫描结果页使用此组件
- [ ] 前 3 名排名序号金色高亮
- [ ] 点击行跳转个股详情
