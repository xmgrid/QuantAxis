# Agent F · UI 框架与导航

> **依赖**：Agent A（ConfigDao）  
> **输出给**：所有 Agent（提供 ThemeConfig, 导航框架, 通用组件）  
> **工期**：1.5 周

---

## 任务清单

### F1. 主题系统

**文件**：`lib/config/theme_config.dart`

5 套主题，全部使用 CSS 变量式定义：

```dart
class AppTheme {
  final String name;
  final Color phoneBg;       // 主背景
  final Color sheet;         // 卡片背景
  final Color card;          // 次级卡片
  final Color separator;     // 分割线
  final Color text;          // 主文字
  final Color textSecondary; // 次级文字
  final Color textTertiary;  // 三级文字
  final Color red;           // 涨
  final Color green;         // 跌
  final Color blue;          // 强调
  final Color accent;        // 高亮
}

class ThemeConfig {
  static const themes = {
    'dark': AppTheme(
      name: '深空黑',
      phoneBg: Color(0xFF000000),
      sheet: Color(0xFF1C1C1E),
      card: Color(0xFF2C2C2E),
      separator: Color(0xFF38383A),
      text: Color(0xFFFFFFFF),
      textSecondary: Color(0xFF98989D),
      textTertiary: Color(0xFF636366),
      red: Color(0xFFFF453A),
      green: Color(0xFF30D158),
      blue: Color(0xFF0A84FF),
      accent: Color(0xFFFF9F0A),
    ),
    'midnight': AppTheme(/* 午夜蓝 #0a0e27 */),
    'charcoal': AppTheme(/* 炭灰 #1a1a1a */),
    'light': AppTheme(/* 亮白 #f2f2f7 */),
    'warm': AppTheme(/* 暖棕 #1c1814 */),
  };

  static AppTheme current;
  static void switchTo(String name) { /* 更新 + 通知监听者 */ }
}
```

**验收**：
- [ ] 5 套主题一键切换
- [ ] 所有 Widget 通过 `ThemeConfig.current` 获取颜色
- [ ] K 线图颜色随主题联动（通过重绘触发）

---

### F2. 应用入口 + Tab 导航

**文件**：`lib/main.dart`
```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await QuantAxisDB.instance.initialize();
  await ConfigDao().initDefaults();
  runApp(QuantAxisApp());
}
```

**文件**：`lib/app.dart`
```dart
class QuantAxisApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider(
      create: (_) => AppState(),
      child: MaterialApp(
        theme: ThemeData.dark().copyWith(/* 从 ThemeConfig 读取 */),
        home: MainShell(),
      ),
    );
  }
}
```

**文件**：`lib/screens/main_shell.dart`

底部 Tab 栏结构：
```
[市场] [选股] [个股] [跟踪]
```

```dart
class MainShell extends StatefulWidget {}

class _MainShellState extends State<MainShell> {
  int _currentTab = 0;
  String _currentSub = 'dashboard'; // dashboard / screener / portfolio

  static const _tabs = [
    ('市场', Icons.show_chart),      // 0
    ('选股', Icons.search),          // 1
    ('个股', Icons.candlestick_chart),// 2
    ('跟踪', Icons.account_balance), // 3
  ];

  // 版本控制：V0.5 隐藏选股+跟踪，V1.0 隐藏跟踪
  List<bool> get _tabVisibility {
    switch (version) {
      case 'v05': return [true, false, true, false];
      case 'v10': return [true, true,  true, false];
      default:    return [true, true,  true, true];
    }
  }

  Widget _buildBody() {
    switch (_currentTab) {
      case 0: return DashboardScreen(subPage: _currentSub);
      case 1: return ScreenerScreen();
      case 2: return StockDetailScreen(tsCode: _lastViewedStock);
      case 3: return PortfolioScreen();
      default: return DashboardScreen();
    }
  }
}
```

**验收**：
- [ ] Tab 切换流畅
- [ ] V0.5/V1.0 版本下 Tab 可见性正确
- [ ] 个股 Tab 保持上次查看的股票
- [ ] 深色背景下 Tab 栏清晰

---

### F3. 版本控制

**文件**：`lib/config/app_config.dart`

```dart
enum AppVersion { v05, v10, v15, v20 }

class AppConfig {
  static AppVersion version = AppVersion.v20;  // 开发阶段默认为完整版

  static String get versionLabel {
    switch (version) {
      case AppVersion.v05: return 'V0.5 数据';
      case AppVersion.v10: return 'V1.0 图表';
      case AppVersion.v15: return 'V1.5 选股';
      case AppVersion.v20: return 'V2.0 完整';
    }
  }
}
```

**验收**：
- [ ] 版本切换后 Tab 可见性 + 页面内容联动
- [ ] 版本号显示在设置页底部

---

### F4. 设置页面

**文件**：`lib/screens/settings_screen.dart`

分组结构（参考原型）：

| 分组 | 配置项 |
|------|--------|
| 🔌 数据源 | 数据提供商（Tushare Pro ✓）/ API Token / 自动更新时间 / 连接状态 |
| 📥 股票池 | **导入股票**（种子/手动/CSV）→ / 当前股票数 / 行业覆盖 |
| 📡 K线周期 | 日线 toggle / 周线（合成）/ 月线（合成）/ 分钟线（可循环切换 5/15/30/60） |
| 📊 分析范围 | 股票池数量 / 历史数据年数（点击循环 1/2/3 年）/ 计算线程 |
| 🗂️ 数据 | 数据库大小 / 导出 CSV / 导入数据 / 清空数据库 |

每个设置项用 `ListTile`，点击切换型用 `Switch`，选择型推新页面。

**验收**：
- [ ] 日线 toggle 可开关
- [ ] 分钟线点击循环切换
- [ ] 历史年数点击循环 1→2→3→1
- [ ] 清空数据库有二次确认弹窗

---

### F5. 股票导入流程

**文件**：`lib/screens/import_screen.dart`

三步流程：
1. **选择方式**：行业种子 / 手动添加 / CSV 导入
2. **执行导入**：
   - 种子：行业勾选网格 → 实时预览数量 → 确认导入
   - 手动：搜索栏 → 结果列表 → 点击添加 → 已添加 chip 行 → 确认
   - CSV：文件选择器 → 解析预览表格 → 确认
3. **完成**：引导前往数据源设置或返回首页

```dart
class ImportScreen extends StatefulWidget {}

enum ImportStep { choose, execute, complete }
enum ImportMethod { seed, manual, csv }
```

**验收**：
- [ ] 行业种子勾选后实时更新预计数量
- [ ] 手动搜索支持代码/名称模糊匹配
- [ ] CSV 解析支持缺少可选列
- [ ] 导入完成后 stocks 表正常

---

### F6. 通用 Widget 组件

**文件**：`lib/widgets/stock_row.dart`
- 股票列表行（名称 + 代码 + 价格 + 涨跌幅）

**文件**：`lib/widgets/index_card.dart`
- 指数卡片（名称 + 数值 + 涨跌幅）

**文件**：`lib/widgets/strategy_card.dart`
- 策略卡片（名称 + 描述 + 分类·标签 + 匹配数 + 左侧色条）

**文件**：`lib/widgets/signal_tag.dart`
- 信号标签 chip（文字 + 高亮/普通态）

**文件**：`lib/utils/toast.dart`
- 底部 Toast 提示（2 秒自动消失）

**验收**：
- [ ] 所有组件支持深色/浅色主题
- [ ] Toast 不阻塞交互
