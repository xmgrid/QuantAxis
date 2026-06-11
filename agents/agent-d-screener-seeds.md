# Agent D · 选股扫描 + Vibe 2.0 种子策略

> **依赖**：Agent A (StrategyDao, VibeResultDao), Agent B (KlinePainter), Agent C (StrategyRunner, VibeEngine)  
> **输出给**：Agent G  
> **工期**：2 周（+0.5 周 Vibe 2.0 管线UI）

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

展示 Vibe 2.0 五大算法模块（参考原型 `screen-triple`）：

#### 种子一 · 趋势策略 (Trend Following)
- **算法**：双均线多头排列三重确认
- **公式**：`close > MA20` AND `MA20 > MA60` AND `close > close[5日前]`
- **参数标签**：MA快=20日 · MA慢=60日 · 短期回看=5日 · 最少60日数据
- **可配置**：MA周期、回看天数、三个条件开关 — 全部支持滑块/Toggle调整
- **⚙️ 参数面板**：展开显示当前默认值 → 「📋 克隆并自定义」进入完整参数编辑
- 匹配数量 + 「▶ 查看结果」按钮

#### 种子二 · 行业轮动策略 (Sector Rotation)
- **算法**：每行业选20日收益率最高龙头，仅纳入 ret_20 > 0 正收益标的
- **公式**：`max(行业内所有个股 ret_20)`, filter `ret_20 > 0`
- **参数标签**：动量周期=20日 · 最低收益=0% · 每行业上限=1只
- **可配置**：动量周期、最低收益门槛、行业上限数
- 当前强势行业展示
- 匹配数量 + 「▶ 查看结果」按钮

#### 种子三 · 多因子评分策略 (Multi-Factor Scoring)
- **算法**：综合评分排名取 Top N
- **公式**：`Score = 50 + ret_20 × 1.5 − vol × 2.0`
- **参数标签**：动量周期=20日 · 基准分=50 · 动量权重=1.5 · 波动惩罚=2.0 · Top N=20
- **可配置**：基准分、权重系数、波动惩罚、Top N数量、最少数据天数
- 匹配数量 + 「▶ 查看结果」按钮

#### 三策略精选 + 七条件过滤（合并卡片）
- **精选公式**：`趋势集 ∩ 轮动集 ∩ 因子集`
- **七条件**：非ST + 10日涨幅 + 量比 + 涨幅区间 + 收盘>MA5 + 资金流入 + 超额成交
- **双模式**：严格/宽松 — 一键切换，阈值联动更新
- **计数展示**：交集X只 · 7条件宽松Y只 · 7条件严格Z只
- 「▶ 运行 Vibe 2.0 完整分析」按钮

```dart
class VibeSeedStrategyCard extends StatelessWidget {
  final String title;
  final String algorithmFormula;      // 核心公式（一行）
  final List<String> conditionTags;   // 算法标签
  final Map<String, String> paramDefaults; // 参数名→默认值
  final int matchCount;
  final Color accentColor;
  final VoidCallback onViewResults;
  final VoidCallback onCloneAndCustomize;  // → 打开参数编辑页
}
```

**Vibe 2.0 种子策略加载**：
```dart
// 从 StrategyDao 加载，而非硬编码 DSL
final vibeSeeds = await StrategyDao().getSeedStrategies();
// vibeSeeds 包含4条：趋势、轮动、因子、完整管线
// 每条策略的 configJson 包含全部可配置参数
// 用户不可直接编辑种子，通过 cloneStrategy() 创建副本后调整
```

**验收**：
- [ ] 三个种子策略卡片显示实际算法公式（非通用描述）
- [ ] 每个策略卡有 ⚙️ 参数展开面板，显示当前默认值
- [ ] 三策略交集 + 七条件过滤 合并卡片，含严格/宽松模式切换
- [ ] 切换严格/宽松时阈值标签联动更新
- [ ] 「克隆并自定义」→ 进入参数编辑页，所有参数可独立调整
- [ ] 克隆后保存 → strategies 表 source_strategy_id 正确追溯
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
