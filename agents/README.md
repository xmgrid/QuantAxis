# QuantAxis · Multi-Agent Development Plan

> 基于 `产品原型.html` 和 `产品设计方案.html` 的任务拆解  
> 每个 Agent 独立可执行，文件自包含，明确输入/输出/依赖/验收标准

---

## 架构总览

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Agent A  │  │ Agent B  │  │ Agent C  │  │ Agent D  │
│ 数据层   │  │ K线+指标 │  │ 策略引擎 │  │ 选股扫描 │
└────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │             │
     └─────────────┴──────┬──────┴─────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
     ┌────┴─────┐   ┌────┴─────┐   ┌────┴─────┐
     │ Agent E  │   │ Agent F  │   │ Agent G  │
     │持仓+买卖点│   │ UI框架   │   │集成+收尾 │
     └──────────┘   └──────────┘   └──────────┘
```

## 技术栈

| 层 | 选型 |
|----|------|
| 框架 | Flutter 3.x + Dart |
| 本地数据库 | sqflite (SQLite) |
| 状态管理 | Provider / Riverpod |
| HTTP | dio |
| K线渲染 | CustomPainter (Canvas) |
| 后台任务 | Dart Isolate |

## Agent 列表

| Agent | 文件名 | 职责 | 预估工期 |
|-------|--------|------|----------|
| A | `agent-a-data-layer.md` | SQLite 建表、Tushare 接入、数据同步、导入导出 | 2 周 |
| B | `agent-b-kline-indicators.md` | K 线 Canvas 渲染、技术指标计算引擎（MA/MACD/KDJ/RSI/BOLL/VOL）| 2 周 |
| C | `agent-c-strategy-engine.md` | 策略 DSL 解析、条件扫描、本地回测、策略编辑器 | 2 周 |
| D | `agent-d-screener-seeds.md` | 选股页面、行业筛选、种子策略（趋势/轮动/多因子）、扫描 Top N | 1.5 周 |
| E | `agent-e-portfolio-bs.md` | 模拟持仓、收益曲线、买卖点分析（量能+换手率+支撑阻力）| 1.5 周 |
| F | `agent-f-ui-shell.md` | 应用框架（Tab 导航、主题切换、设置页、股票导入流程）| 1.5 周 |
| G | `agent-g-integration.md` | 集成联调、版本迭代控制（V0.5→V2.0）、性能优化、测试 | 1 周 |

## 依赖关系

```
A (数据层) ──┬── B (K线+指标)
             ├── C (策略引擎)
             ├── E (持仓+买卖点)
             └── F (UI框架) ──┬── B
                              ├── C ── D (选股扫描)
                              └── E
G (集成) 依赖所有 Agent 完成
```

## 给 Agent 的通用规范

1. **命名**：Dart 文件 `snake_case`，类 `PascalCase`，Widget `PascalCase`
2. **注释**：每个公开方法必须有文档注释
3. **深色主题**：从 `ThemeConfig` 获取颜色，禁止硬编码色值
4. **错误处理**：所有 I/O 操作 try-catch，用户可见错误用 `Toast` 展示
5. **性能**：列表用 `ListView.builder`，大计算放 Isolate
6. **验收**：每个 Agent 任务末尾有验收清单，完成后逐项打勾

---

## 数据模型速查

```dart
// 股票基础信息
class Stock {
  String tsCode;     // 000001.SZ
  String symbol;     // 000001
  String name;       // 平安银行
  String industry;   // 银行
  bool enabled;      // 是否纳入分析
}

// 日线行情
class DailyQuote {
  String tsCode;
  DateTime tradeDate;
  double open, high, low, close, preClose, change, pctChg;
  double vol, amount;
}

// 分钟线行情（可选）
class MinuteQuote {
  String tsCode;
  DateTime tradeTime;
  String period;     // 5/15/30/60
  double open, high, low, close, vol, amount;
}

// 策略定义
class Strategy {
  int id;
  String name;
  String category;   // technical / fundamental / custom / seed
  String conditionsJson; // DSL JSON
}

// 扫描结果
class ScanResult {
  int strategyId;
  String tsCode;
  DateTime tradeDate;
  double matchScore;
  List<String> matchedConditions;
}

// 模拟持仓
class MockPosition {
  String tsCode;
  double shares;
  double avgCost;
  DateTime buyDate;
}
```

---

## 目录结构规范

```
lib/
├── main.dart                    # 入口
├── app.dart                     # MaterialApp + 主题
├── config/
│   ├── theme_config.dart        # 5 套主题定义
│   └── app_config.dart          # 版本控制 + 常量
├── models/
│   ├── stock.dart
│   ├── quote.dart
│   ├── strategy.dart
│   └── portfolio.dart
├── database/
│   ├── database_helper.dart     # SQLite 初始化 + 迁移
│   ├── stock_dao.dart
│   ├── quote_dao.dart
│   └── strategy_dao.dart
├── datasource/
│   ├── datasource_interface.dart
│   ├── tushare_adapter.dart     # Tushare Pro API
│   └── csv_importer.dart
├── engine/
│   ├── indicator_calc.dart      # MA/MACD/KDJ/RSI/BOLL/VOL
│   ├── strategy_parser.dart     # DSL → 条件树
│   ├── strategy_runner.dart     # 遍历股票池执行策略
│   └── backtest_engine.dart     # 回测
├── charts/
│   ├── kline_painter.dart       # K线 CustomPainter
│   ├── indicator_painter.dart   # 副图指标
│   └── equity_painter.dart      # 收益曲线
├── screens/
│   ├── dashboard_screen.dart
│   ├── stock_detail_screen.dart
│   ├── screener_screen.dart
│   ├── portfolio_screen.dart
│   ├── portfolio_detail_screen.dart
│   ├── settings_screen.dart
│   ├── import_screen.dart
│   ├── strategy_editor_screen.dart
│   ├── seed_strategies_screen.dart
│   └── scan_screen.dart
├── widgets/
│   ├── stock_row.dart
│   ├── index_card.dart
│   ├── strategy_card.dart
│   ├── signal_tag.dart
│   ├── bs_analysis_card.dart
│   ├── volume_analysis_bar.dart
│   └── turnover_analysis_bar.dart
└── utils/
    ├── toast.dart
    ├── formatter.dart
    └── isolate_runner.dart
```
