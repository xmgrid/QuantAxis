# Agent B · K线图表 + 技术指标引擎

> **依赖**：Agent A（数据层 — StockDao, QuoteDao）  
> **输出给**：Agent D, E  
> **工期**：2 周

---

## 任务清单

### B1. 技术指标计算引擎

**文件**：`lib/engine/indicator_calc.dart`

所有计算用纯 Dart 实现，不依赖原生库。输入 `List<DailyQuote>`，输出对应指标数组。

```dart
class IndicatorResult {
  final List<double?> values;    // 与输入 quotes 等长，null 表示该位置无计算值
  final List<int> crossUp;       // 金叉位置索引
  final List<int> crossDown;     // 死叉位置索引
}

class IndicatorCalc {
  /// MA 均线
  static List<double?> ma(List<DailyQuote> quotes, int period);

  /// EMA 指数均线（MACD 基础）
  static List<double?> ema(List<DailyQuote> quotes, int period);

  /// MACD: 返回 DIF, DEA, MACD柱
  static ({List<double?> dif, List<double?> dea, List<double?> macd})
    macd(List<DailyQuote> quotes, {int fast=12, int slow=26, int signal=9});

  /// KDJ: 返回 K, D, J
  static ({List<double?> k, List<double?> d, List<double?> j})
    kdj(List<DailyQuote> quotes, {int n=9, int m1=3, int m2=3});

  /// RSI
  static List<double?> rsi(List<DailyQuote> quotes, {int period=14});

  /// BOLL 布林带: 返回 mid, upper, lower
  static ({List<double?> mid, List<double?> upper, List<double?> lower})
    boll(List<DailyQuote> quotes, {int period=20, double k=2.0});

  /// VOL 成交量均线
  static List<double?> volMa(List<DailyQuote> quotes, int period);

  /// 量比
  static double volRatio(List<DailyQuote> quotes, {int period=5});

  /// ADX 趋势强度（种子趋势策略用）
  static List<double?> adx(List<DailyQuote> quotes, {int period=14});

  /// 换手率分析
  static ({double today, double avg5, double avg20, String level}) 
    turnoverAnalysis(List<DailyQuote> quotes);

  /// 通用：找金叉死叉
  static List<int> findCross(List<double?> fast, List<double?> slow, {bool up=true});
}
```

**换手率等级定义**：
```dart
String turnoverLevel(double rate) {
  if (rate < 1.0) return '冷清';
  if (rate < 3.0) return '正常';
  if (rate < 7.0) return '活跃';
  return '过热';
}
```

**验收**：
- [ ] MA/EMA 与 Tushare 数据对比误差 < 0.1%
- [ ] MACD/KDJ/RSI/BOLL 与同花顺对比误差 < 1%
- [ ] 1200 条日线数据全套指标计算 < 10ms
- [ ] 空列表 / 不足周期数据的边界情况有处理（返回 null）

---

### B2. K 线图 CustomPainter

**文件**：`lib/charts/kline_painter.dart`

```dart
class KlinePainter extends CustomPainter {
  final List<DailyQuote> quotes;
  final String period;               // day/week/month/60min
  final Set<String> mainIndicators;  // {'MA5','MA10','MA20','MA60','BOLL'}
  final Set<String> subIndicators;   // {'MACD','KDJ','RSI','VOL'}
  final List<KlineMarker> markers;   // B/S 标记
  final ({double support1, double support2, double resist1})? levels;

  // 手势交互
  final bool showCrosshair;          // 十字光标
  final Offset? crosshairPosition;
  final double scaleX;               // 缩放
  final double offsetX;              // 平移
}
```

**绘制顺序**：
1. 背景网格 (4 条水平线 + 价格标签)
2. 支撑位/阻力位虚线（蓝色 / 金色）
3. 蜡烛图（红涨绿跌，空心/实心）
4. 主图指标线（MA 彩色线 + BOLL 通道）
5. B/S 标记圆点（B=红色 S=绿色，白色字体）
6. 十字光标（可选，长按触发）

**手势交互**（嵌入到父 Widget 的 GestureDetector）：
- 单指拖拽 → 平移
- 双指缩放 → scaleX 变化
- 长按 → 显示十字光标 + 浮层详情
- 双击 → 重置缩放

**KlineMarker 结构**：
```dart
class KlineMarker {
  final int index;       // 在 quotes 中的位置
  final String label;    // 'B' or 'S'
  final Color color;
  final String reason;   // 'KDJ金叉' etc.
}
```

**验收**：
- [ ] 50 根蜡烛图渲染 < 16ms（60fps）
- [ ] 缩放平移流畅无卡顿
- [ ] 十字光标显示 OHLC 四价
- [ ] B/S 标记位置准确
- [ ] 深色/浅色主题切换重绘正确

---

### B3. 副图指标绘制

**文件**：`lib/charts/indicator_painter.dart`

每个副图指标占用 80-120px 高度：

```dart
class IndicatorPainter extends CustomPainter {
  final String indicatorType;  // 'MACD','KDJ','RSI','VOL'
  final List<DailyQuote> quotes;
  final double width;
  final double height;
}
```

**各副图绘制规范**：
| 指标 | 绘制内容 | 颜色 |
|------|----------|------|
| MACD | DIF线(蓝) + DEA线(橙) + 柱状图(红涨绿跌) | 蓝/橙/红绿 |
| KDJ | K线(白) + D线(黄) + J线(紫) + 20/80 超卖超买线 | 白/黄/紫 |
| RSI | RSI 线(蓝) + 30/70 参考线 | 蓝 |
| VOL | 成交量柱(红涨绿跌) + MA5/MA20 量均线 | 红绿/蓝/橙 |

**验收**：
- [ ] 四种副图指标全部可切换
- [ ] 主图+副图同时滚动同步
- [ ] 指标值域超出范围时自动缩放 Y 轴

---

### B4. 个股详情页

**文件**：`lib/screens/stock_detail_screen.dart`

页面结构（自上而下，参考原型）：
1. 顶部返回按钮
2. 股票名称 + 代码 + 自选 badge
3. 当前价（36pt 超轻字重）+ 涨跌额/涨跌幅
4. 周期切换：分时/日K/周K/月K/60分
5. **K 线图**（200px）+ 指标切换
6. 副图指标区（80px，可展开/收起）
7. 今开/最高/最低/成交量/成交额/换手率（grid）
8. **🎯 买卖点分析卡片**（Agent E 提供 Widget）

```dart
class StockDetailScreen extends StatefulWidget {
  final String tsCode;
}

class _StockDetailScreenState extends State<StockDetailScreen> {
  String _activePeriod = 'day';
  String _activeIndicator = 'MACD';
  List<DailyQuote> _quotes = [];
  // ...

  Future<void> _loadData() async {
    _quotes = await QuoteDao().getQuotes(widget.tsCode,
      from: DateTime.now().subtract(Duration(days: 365 * historyYears)));
  }
}
```

**验收**：
- [ ] 周期切换重绘即时响应（< 100ms）
- [ ] 指标切换即时响应
- [ ] 加载 1200 条数据 + 渲染 < 500ms
- [ ] 横竖屏旋转不丢失状态
