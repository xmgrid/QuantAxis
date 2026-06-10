# Agent E · 持仓管理 + 买卖点分析

> **依赖**：Agent A (QuoteDao), Agent B (IndicatorCalc, KlinePainter), Agent C (StrategyRunner)  
> **输出给**：Agent G  
> **工期**：1.5 周

---

## 任务清单

### E1. 模拟持仓管理

**文件**：`lib/screens/portfolio_screen.dart`

页面结构（参考原型）：
1. **资产概览卡片**：
   - 模拟总市值（42pt 超轻字重）
   - 当日盈亏 / 累计盈亏 / 仓位比例
   - 红涨绿跌

2. **持仓列表**：
   - 股票名称 + 股数 + 成本价
   - 现价 + 涨跌幅
   - 点击进入持仓详情

3. **「📊 查看详细分析」按钮** → 进入 E2

4. **录入方式**：
   - 手动录入：股票代码 + 数量 + 成本 + 日期
   - CSV 批量导入

```dart
class PortfolioScreen extends StatefulWidget {}

class _PortfolioScreenState extends State<PortfolioScreen> {
  List<MockPosition> _positions = [];
  double _totalValue = 0;
  double _dailyPnl = 0;
  double _totalPnl = 0;

  Future<void> _loadPortfolio() async {
    _positions = await PositionDao().getAll();
    // 对每个持仓，查最新价格计算盈亏
    for (var p in _positions) {
      final latest = await QuoteDao().getLatest(p.tsCode);
      if (latest != null) {
        p.currentPrice = latest.close;
        p.pnl = (latest.close - p.avgCost) * p.shares;
      }
    }
    _totalValue = _positions.fold(0, (s, p) => s + p.currentPrice * p.shares);
    _totalPnl = _positions.fold(0, (s, p) => s + p.pnl);
  }
}
```

**验收**：
- [ ] 手动添加持仓成功
- [ ] 盈亏计算准确（含手续费）
- [ ] 涨跌颜色正确（红涨绿跌）
- [ ] CSV 导入 10 条持仓记录成功

---

### E2. 持仓详情页

**文件**：`lib/screens/portfolio_detail_screen.dart`

页面结构（参考原型）：
1. 资产概览（同 E1，加累计收益率）
2. **📈 收益曲线**：
   - Canvas 绘制蓝色面积图
   - X 轴日期 / Y 轴市值
   - 叠加沪深 300 基准线

3. **📊 行业配置**：
   - 水平条形图
   - 食品饮料 63% / 电力设备 19% / 半导体 18%

4. **📝 交易记录**：
   - 买入/卖出 标记
   - 股票名称 + 数量@价格 + 日期

```dart
class PortfolioDetailScreen extends StatefulWidget {}

// 收益曲线 CustomPainter
class EquityPainter extends CustomPainter {
  final List<({DateTime date, double value})> equityCurve;
  final List<({DateTime date, double value})> benchmarkCurve;
}
```

**验收**：
- [ ] 收益曲线平滑渲染
- [ ] 行业配置柱状图比例正确
- [ ] 基准对比线可见

---

### E3. 买卖点分析卡片（核心）

**文件**：`lib/widgets/bs_analysis_card.dart`

这是个股详情页底部的买卖点分析卡片，直接嵌入 `StockDetailScreen`。

```dart
class BSAnalysisCard extends StatelessWidget {
  final String tsCode;
  final List<DailyQuote> quotes;  // 最近的行情数据
}
```

**卡片结构**（参考原型，6 个区块）：

#### 区块 1：信号统计（5 栏横排）
```dart
Row(
  children: [
    _SignalStat(label: '买入', value: '${buySignals.length}', color: red),
    _SignalStat(label: '卖出', value: '${sellSignals.length}', color: green),
    _SignalStat(label: '量能', value: '${volRatio}x', color: blue),
    _SignalStat(label: '换手', value: '${turnover}%', color: accent),
    _SignalStat(label: '评分', value: '$score', color: accent),
  ],
)
```

#### 区块 2：成交量分析
- 今日成交量 + 量比 + 20 日均量
- 量能柱（65% 位置高亮）
- 缩量—均量—放量 刻度

#### 区块 3：换手率分析
- 今日 / 5日均 / 20日均
- 换手率柱（冷清<1% 正常1-3% 活跃3-7% 过热>7%）
- 趋势描述（如 "连续3日攀升 · 市场关注度升温"）

#### 区块 4：买入信号明细
```dart
// 自动检测以下信号
class BuySellDetector {
  static List<Signal> detectBuy(List<DailyQuote> quotes) {
    List<Signal> signals = [];
    
    // KDJ 金叉 + 量比>1.5
    if (kdjCrossUp && volRatio > 1.5) signals.add(Signal('KDJ金叉+放量确认', '强买入'));
    
    // MA5 上穿 MA20 + 成交量连续放大
    if (maCrossUp && volIncreasing(3)) signals.add(Signal('MA5突破MA20+量增', '买入'));
    
    // 突破前高 + 巨量
    if (breakout && volRatio > 2.5) signals.add(Signal('箱体突破+巨量', '突破买入'));
    
    return signals;
  }
  
  static List<Signal> detectSell(List<DailyQuote> quotes) {
    List<Signal> signals = [];
    
    // RSI 超买 + 量价背离
    if (rsi > 70 && volDeclining()) signals.add(Signal('RSI超买+量价背离', '减仓'));
    
    // 高位缩量滞涨
    if (highPosition && volShrinking(3)) signals.add(Signal('高位缩量滞涨', '观察'));
    
    return signals;
  }
}
```

#### 区块 5：卖出/警示信号明细

#### 区块 6：关键价位
```dart
Row(
  children: [
    _PriceLevel(label: '阻力位', price: resist1, desc: '前高+布林上轨', color: accent),
    _PriceLevel(label: '当前价', price: currentPrice, desc: '量比 ${volRatio}x'),
    _PriceLevel(label: '支撑位1', price: support1, desc: 'MA20+密集成交', color: blue),
    _PriceLevel(label: '支撑位2', price: support2, desc: 'MA60+前低', color: blue),
  ],
)
```

**信号检测逻辑**：
```dart
class Signal {
  final String description;  // 'KDJ金叉'
  final String level;        // '强买入' / '买入' / '减仓' / '观察'
  final Color color;         // red / green / accent
  final String detail;       // 'K=42.3 ↑ D=38.6 · 量比2.3x · 06-09'
  final SignalType type;     // buy / sell / warning
}
```

**验收**：
- [ ] 至少检测 4 种买入信号
- [ ] 至少检测 2 种卖出/警示信号
- [ ] 量能+换手率数据准确
- [ ] 支撑位/阻力位计算合理
- [ ] 卡片在深色/浅色主题下正确渲染

---

### E4. 买卖点 K 线标注

**文件**：`lib/widgets/kline_marker_overlay.dart`

在 K 线图上叠加 B/S 标记和支撑阻力线：
- B 标记（红色实心圆 + 白色 B 字）
- S 标记（绿色实心圆 + 白色 S 字）
- 支撑位虚线（蓝色虚线）
- 阻力位虚线（金色虚线）

这些已由 Agent B 的 `KlinePainter.markers` 和 `KlinePainter.levels` 支持，Agent E 负责生成数据。

```dart
class BSMarkerGenerator {
  static List<KlineMarker> generate(String tsCode, List<DailyQuote> quotes) {
    final signals = BuySellDetector.detectBuy(quotes);
    // 将信号映射到对应的 K 线位置
    return signals.map((s) => KlineMarker(
      index: s.quoteIndex,
      label: s.type == SignalType.buy ? 'B' : 'S',
      color: s.type == SignalType.buy ? Colors.red : Colors.green,
      reason: s.description,
    )).toList();
  }
  
  static ({double support1, double support2, double resist1}) 
    generateLevels(List<DailyQuote> quotes) {
    // support1 = MA20, support2 = min(MA60, 近期低点)
    // resist1 = max(近期高点, BOLL上轨)
  }
}
```
