# Agent G · 集成联调 + Vibe 2.0 收尾

> **依赖**：Agent A, B, C, D, E, F 全部完成  
> **工期**：1.5 周（+0.5 周 Vibe 2.0 集成验证）

---

## 任务清单

### G1. 全局状态管理

**文件**：`lib/providers/app_state.dart`

```dart
class AppState extends ChangeNotifier {
  AppVersion _version = AppVersion.v20;
  String _theme = 'dark';
  int _stockCount = 0;
  double _dbSize = 0;

  // 股票池变更通知
  Future<void> refreshStockCount() async {
    _stockCount = await StockDao().getCount();
    notifyListeners();
  }

  // 主题切换 → 通知所有监听者重绘
  void setTheme(String name) {
    _theme = name;
    ThemeConfig.switchTo(name);
    notifyListeners();
  }
}
```

**验收**：
- [ ] Provider 树正确，所有页面可访问 AppState
- [ ] 主题切换全局生效
- [ ] 股票池变更后 Dashboard 自动刷新

---

### G2. 页面路由串联

确保以下页面跳转链路完整：

```
Dashboard
  ├── 搜索 → StockDetail
  ├── 指数卡片 → StockDetail
  ├── 分类行 → Screener (带行业筛选)
  ├── 强势股票行 → StockDetail
  ├── 策略卡片 → SeedStrategies / StrategyResults
  └── 设置按钮 → Settings

Screener
  ├── 种子策略卡片 → SeedStrategies
  ├── 策略卡片 → StrategyResults → StockDetail
  ├── 扫描按钮 → ScanScreen → StockDetail
  └── 创建策略 → StrategyEditor

StockDetail
  ├── 返回 → 上一页
  └── 分析策略信号 → 无跳转（同页展示）

Portfolio
  ├── 持仓行 → PortfolioDetail
  └── 详细分析 → PortfolioDetail

Settings
  ├── 导入股票 → ImportScreen
  └── 导出 CSV → 系统分享菜单
```

**验收**：
- [ ] 所有跳转无 crashed 路由
- [ ] 返回栈正确（不会回到错误页面）
- [ ] Deep link 情况下 Tab 状态恢复

---

### G3. 版本迭代开关验证

验证每个版本的功能边界：

| 功能 | V0.5 | V1.0 | V1.5 | V2.0 |
|------|------|------|------|------|
| 数据同步 | ✓ | ✓ | ✓ | ✓ |
| 股票池管理 | ✓ | ✓ | ✓ | ✓ |
| 股票导入 | ✓ | ✓ | ✓ | ✓ |
| 大盘仪表盘 | ✗ | ✓ | ✓ | ✓ |
| K线图表 | ✗ | ✓ | ✓ | ✓ |
| 技术指标 | ✗ | ✓ | ✓ | ✓ |
| 选股策略 | ✗ | ✗ | ✓ | ✓ |
| **Vibe 2.0 种子策略** | ✗ | ✗ | ✓ | ✓ |
| **Vibe 三策略精选(交集)** | ✗ | ✗ | ✗ | ✓ |
| **Vibe 七条件过滤** | ✗ | ✗ | ✗ | ✓ |
| **Vibe 参数配置页** | ✗ | ✗ | ✗ | ✓ |
| 策略编辑器 | ✗ | ✗ | ✗ | ✓ |
| 模拟持仓 | ✗ | ✗ | ✓ | ✓ |
| 买卖点分析 | ✗ | ✓ | ✓ | ✓ |
| **Vibe 信号评分卡** | ✗ | ✗ | ✗ | ✓ |
| **策略克隆+自定义** | ✗ | ✗ | ✗ | ✓ |

**验收**：
- [ ] V0.5 选股/跟踪 Tab 隐藏
- [ ] V1.0 跟踪 Tab 隐藏
- [ ] V1.5 策略编辑器入口隐藏
- [ ] V1.5 Vibe 三策略+交集可用，七条件过滤不可用
- [ ] V2.0 Vibe 完整管线（含七条件过滤+参数配置）可用
- [ ] 版本切换无需重启
- [ ] 种子策略参数升级不影响用户自定义策略

---

### G3b. Vibe 2.0 种子策略升级流程

当 App 从 V1.5 升级到 V2.0 时，种子策略参数可能新增。升级流程：

```
1. 读取 strategies 表所有 is_seed=1 的策略
2. 检查 schema_version < currentSchemaVersion
3. 对于需要升级的种子策略：
   a. 备份旧 config_json 到内存
   b. 调用 ConfigMigration.migrate() 升级参数格式
   c. 更新 config_json + schema_version
   d. 对于 is_seed=0 的用户自定义策略：
      - 不自动升级（保护用户自定义参数）
      - 在策略列表标记「可升级」提示
      - 用户手动触发升级
4. 写入 VibeResultDao.cleanOldResults() 清理旧缓存
```

**验收**：
- [ ] V1.5→V2.0 升级后种子策略参数刷新
- [ ] 用户自定义策略不受升级影响
- [ ] 旧格式 config_json 正常加载（向前兼容）
- [ ] 升级后 VibeEngine 使用新参数运行结果正确

---

### G4. 性能优化

**检查项**：
- [ ] 股票列表使用 `ListView.builder`（非一次性渲染全部）
- [ ] K 线图 `shouldRepaint` 正确判断（仅数据变化时重绘）
- [ ] 大计算（策略扫描、回测）放入 Isolate
- [ ] 图片/资源 < 1MB
- [ ] 首屏加载 < 1 秒
- [ ] 数据库查询用索引覆盖（EXPLAIN QUERY PLAN）
- [ ] 内存使用 < 200MB（120 只股票正常使用）

**优化工具**：
```dart
// Flutter DevTools → Performance tab
// 检查 rebuild 次数
// 检查 build 方法耗时
```

---

### G5. 边界情况处理

**数据为空时**：
- 股票池为空 → Dashboard 显示"请先导入股票"
- 某只股票无行情数据 → StockDetail 显示"数据加载中"或"暂无行情数据"
- 策略无匹配结果 → 显示"暂无匹配标的"

**网络异常时**：
- Tushare 超时 → Toast "网络请求超时，请重试"
- Token 无效 → Toast "Token 无效，请前往设置更新"
- API 额度用完 → Toast "今日 API 额度已用完，请明日再试"

**数据异常时**：
- 日线数据缺失日期 → 跳过，不崩溃
- 股票代码格式错误 → 提示正确格式（000001.SZ / 600519.SH）

**验收**：
- [ ] 所有空状态有 UI 提示
- [ ] 所有异常有 Toast 提示
- [ ] 无 unhandled exception

---

### G6. 测试

```dart
// 单元测试
test('IndicatorCalc MA计算正确', () {
  final quotes = [/* 已知数据 */];
  final result = IndicatorCalc.ma(quotes, 5);
  expect(result[4], closeTo(expected, 0.001));
});

test('StrategyParser 解析嵌套 AND/OR', () {
  final dsl = '{"logic":"AND","conditions":[...]}';
  final node = StrategyParser.parse(dsl);
  expect(node.logic, LogicOp.AND);
  expect(node.conditions.length, 2);
});

test('StockDao 搜索支持拼音首字母', () {
  final results = await StockDao().search('ZG');
  expect(results.any((s) => s.name == '中芯国际'), true);
});

// Widget 测试
testWidgets('Dashboard 显示指数卡片', (tester) async {
  await tester.pumpWidget(QuantAxisApp());
  expect(find.text('上证指数'), findsOneWidget);
});
```

**验收**：
- [ ] 核心计算逻辑有单元测试（IndicatorCalc, StrategyParser）
- [ ] DAO 有集成测试
- [ ] 关键页面有 Widget 测试

---

### G7. 打包与发布准备

- [ ] `flutter build apk --release` 成功
- [ ] `flutter build ios --release` 成功
- [ ] 应用图标 + 启动页
- [ ] 隐私政策文案（本地存储声明）
- [ ] 风险提示文案（不构成投资建议）

---

## Agent G 执行顺序

1. 先用 G3 检查所有 Agent 的交付物是否完整
2. 用 G2 串联所有页面路由
3. 用 G1 统一状态管理
4. 用 G4 性能优化
5. 用 G5 边界处理
6. 用 G6 测试
7. 用 G7 打包

---

## 每日站会检查

- 各 Agent 交付物是否在 `lib/` 目录下？
- 是否使用了其他 Agent 提供的接口？
- 是否有未处理的依赖循环？
- 是否有超出设计方案范围的新功能？
- 验收清单是否逐项打勾？
