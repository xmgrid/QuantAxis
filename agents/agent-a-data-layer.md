# Agent A · 数据层

> **依赖**：无（最先执行）  
> **输出给**：Agent B, C, E, F  
> **工期**：2 周

---

## 任务清单

### A1. SQLite 数据库初始化

**文件**：`lib/database/database_helper.dart`

- 创建 `QuantAxisDB` 单例
- `onCreate` 中执行建表 SQL
- 数据库版本号 `1`，预留 `onUpgrade` 迁移逻辑
- 数据库文件路径：`await getDatabasesPath() + '/quantaxis.db'`

**表结构**（与设计方案一致）：

```sql
-- 股票基础信息
CREATE TABLE stocks (
  ts_code TEXT PRIMARY KEY,     -- '000001.SZ'
  symbol TEXT NOT NULL,          -- '000001'
  name TEXT NOT NULL,            -- '平安银行'
  industry TEXT,                 -- 申万行业分类（种子）
  list_date TEXT,
  is_hs INTEGER DEFAULT 0,
  enabled INTEGER DEFAULT 1     -- 1=纳入分析范围
);

-- 分类体系
CREATE TABLE categories (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  parent_id INTEGER,
  is_seed INTEGER DEFAULT 0,   -- 1=系统种子行业, 0=用户自定义
  sort_order INTEGER DEFAULT 0
);

-- 股票-分类 多对多
CREATE TABLE stock_category (
  stock_code TEXT,
  category_id INTEGER,
  PRIMARY KEY (stock_code, category_id)
);

-- 日线行情（核心表）
CREATE TABLE daily_quotes (
  ts_code TEXT NOT NULL,
  trade_date TEXT NOT NULL,     -- '20260610'
  open REAL, high REAL, low REAL, close REAL,
  pre_close REAL, change REAL, pct_chg REAL,
  vol REAL, amount REAL,
  PRIMARY KEY (ts_code, trade_date)
);
CREATE INDEX idx_daily_code_date ON daily_quotes(ts_code, trade_date);
CREATE INDEX idx_daily_date ON daily_quotes(trade_date);

-- 分钟线行情（可选）
CREATE TABLE minute_quotes (
  ts_code TEXT NOT NULL,
  trade_time TEXT NOT NULL,
  period TEXT NOT NULL,         -- '5','15','30','60'
  open REAL, high REAL, low REAL, close REAL,
  vol REAL, amount REAL,
  PRIMARY KEY (ts_code, trade_time, period)
);

-- 指数日线
CREATE TABLE index_daily (
  ts_code TEXT NOT NULL,
  trade_date TEXT NOT NULL,
  close REAL, open REAL, high REAL, low REAL,
  change REAL, pct_chg REAL, vol REAL, amount REAL,
  PRIMARY KEY (ts_code, trade_date)
);

-- 策略存储（Vibe 2.0 更新：支持完整参数配置 + 版本管理 + 种子策略溯源）
CREATE TABLE strategies (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  description TEXT DEFAULT '',
  category TEXT NOT NULL DEFAULT 'custom',     -- 'seed' | 'technical' | 'fundamental' | 'custom'
  is_seed INTEGER DEFAULT 0,                   -- 1=系统内置(只读), 0=用户创建(可编辑)
  source_strategy_id INTEGER,                  -- 克隆来源策略ID, NULL=原创
  config_json TEXT NOT NULL,                   -- 完整策略配置JSON (含trend/rotation/factor/filter/global五组参数)
  schema_version INTEGER DEFAULT 2,            -- JSON配置格式版本号
  app_version TEXT DEFAULT '2.0',              -- 创建/更新时的App版本
  sort_order INTEGER DEFAULT 0,                -- 排序权重（种子策略排前面）
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (source_strategy_id) REFERENCES strategies(id)
);

-- Vibe 2.0 分析结果缓存（三策略交集 + 七条件过滤）
CREATE TABLE vibe_results (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  run_date TEXT NOT NULL,                       -- 运行日期 '20260610'
  ts_code TEXT NOT NULL,
  strategy_type TEXT NOT NULL,                  -- 'trend'/'rotation'/'factor'/'intersection'/'seven_filter'
  match_score REAL,                             -- 综合得分
  ret_20 REAL,                                  -- 20日收益率
  volatility REAL,                              -- 波动率
  conditions_passed TEXT,                       -- 通过的条件列表 JSON array
  seven_mode TEXT,                              -- 'strict'/'loose' (仅seven_filter类型)
  created_at TEXT NOT NULL,
  UNIQUE(run_date, ts_code, strategy_type, seven_mode)
);
CREATE INDEX idx_vibe_date_type ON vibe_results(run_date, strategy_type);

-- 扫描结果缓存
CREATE TABLE scan_results (
  strategy_id INTEGER,
  ts_code TEXT,
  trade_date TEXT,
  match_score REAL,
  matched_conditions TEXT,     -- JSON array
  PRIMARY KEY (strategy_id, ts_code, trade_date)
);

-- 模拟持仓
CREATE TABLE mock_positions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ts_code TEXT NOT NULL,
  shares REAL NOT NULL,
  avg_cost REAL NOT NULL,
  buy_date TEXT,
  notes TEXT
);

-- 应用配置键值对
CREATE TABLE app_config (
  key TEXT PRIMARY KEY,
  value TEXT
);
```

**验收**：
- [ ] 首次启动自动建库
- [ ] 所有表创建成功
- [ ] 索引生效（用 `EXPLAIN QUERY PLAN` 验证）

---

### A2. DAO 层（数据访问对象）

每个 DAO 提供标准 CRUD + 批量操作：

**文件**：`lib/database/stock_dao.dart`
```dart
class StockDao {
  Future<void> insertOrUpdate(Stock stock);
  Future<void> batchInsert(List<Stock> stocks);        // 事务批量插入
  Future<List<Stock>> getAll({bool enabledOnly = true});
  Future<List<Stock>> getByIndustry(String industry);
  Future<List<Stock>> search(String keyword);           // 代码/名称/拼音首字母
  Future<void> setEnabled(String tsCode, bool enabled);
  Future<int> getCount();
}
```

**文件**：`lib/database/quote_dao.dart`
```dart
class QuoteDao {
  Future<void> insertOrUpdate(DailyQuote quote);
  Future<void> batchInsert(List<DailyQuote> quotes);   // 事务 + 冲突替换
  Future<List<DailyQuote>> getQuotes(String tsCode, {DateTime? from, DateTime? to});
  Future<DailyQuote?> getLatest(String tsCode);
  Future<DateTime?> getLastTradeDate(String tsCode);
  // 周线/月线合成
  Future<List<DailyQuote>> getWeeklyQuotes(String tsCode);  // 日线聚合
  Future<List<DailyQuote>> getMonthlyQuotes(String tsCode); // 日线聚合
}
```

**文件**：`lib/database/strategy_dao.dart`
```dart
class StrategyDao {
  Future<int> insert(Strategy s);
  Future<void> update(Strategy s);
  Future<void> delete(int id);
  Future<List<Strategy>> getAll({String? category});
  Future<Strategy?> getById(int id);

  // Vibe 2.0 新增
  Future<List<Strategy>> getSeedStrategies();                   // 获取所有种子策略
  Future<List<Strategy>> getCustomStrategies();                 // 获取用户自定义策略
  Future<List<Strategy>> getBySource(int sourceStrategyId);     // 查找克隆自某策略的所有副本
  Future<Strategy> cloneStrategy(int sourceId, String newName); // 克隆种子→自定义
  Future<void> resetToDefaults(int strategyId);                 // 恢复出厂默认参数
  Future<void> saveScanResults(int strategyId, List<ScanResult> results);
  Future<List<ScanResult>> getScanResults(int strategyId, DateTime date);
}

// Vibe 2.0 新增 DAO
class VibeResultDao {
  Future<void> saveResults(String runDate, String strategyType, List<VibeResult> results);
  Future<List<VibeResult>> getByDate(String runDate, {String? strategyType, String? sevenMode});
  Future<VibeResult?> getByCode(String runDate, String tsCode, String strategyType);
  Future<void> cleanOldResults(int keepDays);  // 清理N天前的旧结果
}
```

**文件**：`lib/database/config_dao.dart`
```dart
class ConfigDao {
  Future<void> set(String key, String value);
  Future<String?> get(String key);
  // 预设默认值
  Future<void> initDefaults(); // data_provider, token, auto_sync_time, history_years, etc.
}
```

**验收**：
- [ ] 批量插入 1000 条日线数据 < 500ms
- [ ] 搜索功能支持代码/名称/拼音首字母
- [ ] 周线/月线聚合结果与 Tushare 原始周月线误差 < 0.5%

---

### A3. Tushare 数据源适配器

**文件**：`lib/datasource/datasource_interface.dart`
```dart
abstract class DataSource {
  Future<bool> testConnection();
  Future<List<Stock>> fetchStockBasic();           // 股票基础信息
  Future<List<DailyQuote>> fetchDailyQuotes({
    required String tsCode,
    required DateTime startDate,
    required DateTime endDate,
  });
  Future<List<MinuteQuote>> fetchMinuteQuotes({
    required String tsCode,
    required String period,   // 5/15/30/60
    required DateTime tradeDate,
  });
  Future<List<DailyQuote>> fetchIndexDaily({
    required String indexCode,
    required DateTime startDate,
    required DateTime endDate,
  });
  Future<int> getApiLimit();  // 剩余调用次数
}
```

**文件**：`lib/datasource/tushare_adapter.dart`
- 实现 `DataSource` 接口
- HTTP POST → `https://api.tushare.pro`
- 请求体格式：
```json
{
  "api_name": "daily",
  "token": "{user_token}",
  "params": {"ts_code": "000001.SZ", "start_date": "20210101", "end_date": "20260610"},
  "fields": "ts_code,trade_date,open,high,low,close,pre_close,change,pct_chg,vol,amount"
}
```
- 错误处理：token 无效 / 额度用完 / 网络超时 → 返回明确错误类型
- 请求频率控制：每分钟最多 200 次（Tushare Pro 限制）

**Tushare API 映射表**：
| 数据 | API | 关键参数 |
|------|-----|----------|
| 股票列表 | `stock_basic` | list_status='L' |
| 日线 | `daily` | ts_code, start_date, end_date |
| 分钟线 | `stk_mins` | ts_code, freq, trade_date |
| 指数日线 | `index_daily` | ts_code, start_date, end_date |
| 财务数据 | `fina_indicator` | ts_code, start_date, end_date |

**验收**：
- [ ] 用真实 Token 测试连接
- [ ] 拉取 1 只股票 3 年日线成功
- [ ] Token 无效时返回明确错误
- [ ] 网络超时 10s 自动重试 1 次

---

### A4. 数据同步服务

**文件**：`lib/datasource/sync_service.dart`
```dart
class SyncService {
  final DataSource dataSource;
  final StockDao stockDao;
  final QuoteDao quoteDao;

  /// 全量同步：拉取股票池所有标的的历史日线
  Future<SyncResult> fullSync({
    required List<String> tsCodes,
    required int historyYears,   // 1/2/3 年
    void Function(double progress)? onProgress,
  });

  /// 增量同步：仅拉取最近 N 天
  Future<SyncResult> incrementalSync({
    void Function(double progress)? onProgress,
  });

  /// 拉取分钟线（仅对已开启的标的）
  Future<SyncResult> syncMinuteData({
    required String period,
    required DateTime date,
  });
}

class SyncResult {
  int successCount;
  int failCount;
  List<String> failedCodes;
  Duration elapsed;
}
```

- `fullSync`：遍历股票池 → 每批 20 只 → 拉取日线 → 批量写入 SQLite
- `incrementalSync`：查每只股票最新日期 → 从该日期拉到今天
- 进度回调百分比，供 UI 显示进度条
- 同步完成后记录 `last_sync_time` 到 `app_config`

**验收**：
- [ ] 120 只股票全量同步 3 年日线 < 5 分钟
- [ ] 增量同步仅拉取缺失日期
- [ ] 单只股票失败不影响其他（独立 try-catch）
- [ ] 进度回调正常工作

---

### A5. 数据导入导出

**文件**：`lib/datasource/csv_importer.dart`
- CSV 解析：支持 `ts_code` 或 `symbol` 列、可选 `name`/`industry` 列
- 冲突处理：已存在的股票 → 更新 name/industry，不重复插入

**文件**：`lib/datasource/csv_exporter.dart`
- 导出股票池到 CSV
- 导出日线数据到 CSV（单只或全部）
- 使用 `share_plus` 调用系统分享菜单

**验收**：
- [ ] 导入含 50 只股票的 CSV 成功
- [ ] 导出 CSV 用 Excel 打开格式正确
- [ ] 分享菜单正常弹出

---

### A7. Vibe 2.0 种子策略数据初始化

**文件**：`lib/datasource/vibe_seeds.dart`

首次启动时，向 `strategies` 表插入三个 Vibe 2.0 种子策略：

```dart
class VibeSeedInitializer {
  static Future<void> initialize(StrategyDao dao) async {
    // 仅在首次启动时执行（检查是否已有种子策略）
    final existing = await dao.getSeedStrategies();
    if (existing.isNotEmpty) return;

    // 1. 趋势策略
    await dao.insert(Strategy(
      name: 'Vibe 2.0 趋势策略',
      description: '双均线多头排列三重确认 — close>MA20 且 MA20>MA60 且 close>close[5日前]',
      category: 'seed',
      isSeed: true,
      configJson: jsonEncode({
        'schema_version': 2,
        'name': 'Vibe 2.0 趋势策略',
        'category': 'seed',
        'is_seed': true,
        'trend': {
          'min_data_days': 60, 'ma_fast': 20, 'ma_slow': 60,
          'short_lookback': 5,
          'require_close_above_ma': true,
          'require_ma_cross': true,
          'require_short_momentum': true,
        },
        'rotation': null,  // 此策略不使用轮动
        'factor': null,    // 此策略不使用因子评分
        'filter': null,    // 此策略不使用七条件
        'global': {'industry_filter': '全部', 'max_stocks': 200},
      }),
      schemaVersion: 2,
      sortOrder: 1,
    ));

    // 2. 行业轮动策略
    await dao.insert(Strategy(
      name: 'Vibe 2.0 行业轮动策略',
      description: '每行业选取20日收益率最高龙头，仅纳入ret_20>0的正收益标的',
      category: 'seed',
      isSeed: true,
      configJson: jsonEncode({
        'schema_version': 2,
        'rotation': {
          'momentum_period': 20, 'min_return': 0.0, 'max_per_industry': 1,
        },
        'global': {'industry_filter': '全部', 'max_stocks': 200},
      }),
      schemaVersion: 2,
      sortOrder: 2,
    ));

    // 3. 多因子评分策略
    await dao.insert(Strategy(
      name: 'Vibe 2.0 多因子评分策略',
      description: 'Score = 50 + ret_20×1.5 − vol×2.0，取Top 20',
      category: 'seed',
      isSeed: true,
      configJson: jsonEncode({
        'schema_version': 2,
        'factor': {
          'momentum_period': 20, 'base_score': 50,
          'momentum_weight': 1.5, 'volatility_penalty': 2.0,
          'top_n': 20, 'min_data_days': 60,
        },
        'global': {'industry_filter': '全部', 'max_stocks': 200},
      }),
      schemaVersion: 2,
      sortOrder: 3,
    ));

    // 4. Vibe 2.0 完整管线（三策略交集+七条件）
    await dao.insert(Strategy(
      name: 'Vibe 2.0 完整分析管线',
      description: '趋势∩轮动∩因子 → 精选 → 七条件过滤（支持严格/宽松双模式）',
      category: 'seed',
      isSeed: true,
      configJson: jsonEncode({
        'schema_version': 2,
        'trend': {'min_data_days': 60, 'ma_fast': 20, 'ma_slow': 60, 'short_lookback': 5, 'require_close_above_ma': true, 'require_ma_cross': true, 'require_short_momentum': true},
        'rotation': {'momentum_period': 20, 'min_return': 0.0, 'max_per_industry': 1},
        'factor': {'momentum_period': 20, 'base_score': 50, 'momentum_weight': 1.5, 'volatility_penalty': 2.0, 'top_n': 20, 'min_data_days': 60},
        'filter': {'min_data_days': 30, 'require_amount_data': true, 'c1_exclude_st': true, 'c2_ret_period': 10, 'c2_strict_min': 5.0, 'c2_loose_min': 0.0, 'c3_vol_period': 5, 'c3_strict_min': 1.5, 'c3_loose_min': 1.0, 'c4_strict_low': 3.0, 'c4_strict_high': 5.0, 'c4_loose_low': 0.0, 'c4_loose_high': 7.0, 'c5_ma_period': 5, 'c6_recent_days': 3, 'c6_prior_days': 3, 'c7_amt_period': 5},
        'global': {'industry_filter': '全部', 'max_stocks': 200, 'only_all_three': false, 'strict_mode': false, 'scan_mode': '行业'},
      }),
      schemaVersion: 2,
      sortOrder: 0,  // 排最前面
    ));
  }
}
```

**验收**：
- [ ] 首次启动后 strategies 表包含 4 条种子策略
- [ ] `is_seed=1` 标记正确
- [ ] `config_json` 包含全部五组参数（trend/rotation/factor/filter/global）
- [ ] 完整管线策略的 sortOrder=0 排最前
- [ ] `cloneStrategy()` 能正确创建用户副本（source_strategy_id 追溯来源）

**文件**：`lib/datasource/industry_seeds.dart`

- 硬编码 9 个申万一级行业及其代表股票
- 首次启动时自动写入 `categories` 和 `stocks` 表
- 标记 `is_seed = 1`

```dart
const industrySeeds = {
  '半导体': ['688981.SH','002371.SZ','603501.SH','603986.SH','300782.SZ',...],
  '食品饮料': ['600519.SH','000858.SZ','600887.SH','603288.SH',...],
  '电力设备': ['300750.SZ','601012.SH','300274.SZ','600438.SH',...],
  '医药生物': ['603259.SH','600276.SH','300760.SZ','300015.SZ',...],
  '银行': ['600036.SH','601398.SH','601939.SH','601166.SH',...],
  '汽车': ['002594.SZ','601633.SH','000625.SZ','600104.SH',...],
  '计算机': ['688111.SH','600588.SH','600570.SH','002410.SZ',...],
  '有色金属': ['601899.SH','603993.SH','600111.SH','002460.SZ',...],
  '房地产': ['000002.SZ','600048.SH','001979.SZ','600383.SH',...],
};
```

**验收**：
- [ ] 首次启动后 stocks 表包含 ~150 只种子股票
- [ ] categories 表包含 9 个种子行业
- [ ] `is_seed=1` 标记正确

---

## 输出交付物

```
lib/
├── models/
│   ├── stock.dart
│   ├── quote.dart
│   └── strategy.dart
├── database/
│   ├── database_helper.dart
│   ├── stock_dao.dart
│   ├── quote_dao.dart
│   ├── strategy_dao.dart
│   └── config_dao.dart
├── datasource/
│   ├── datasource_interface.dart
│   ├── tushare_adapter.dart
│   ├── sync_service.dart
│   ├── csv_importer.dart
│   ├── csv_exporter.dart
│   └── industry_seeds.dart
```

## Agent B/C/D/E/F 如何读取 Agent A 的输出

```dart
// 获取数据库实例
final db = QuantAxisDB.instance;

// 获取股票列表
final stocks = await StockDao().getAll(enabledOnly: true);

// 获取某只股票的日线
final quotes = await QuoteDao().getQuotes('000001.SZ',
  from: DateTime(2023, 6, 10), to: DateTime(2026, 6, 10));

// 获取配置
final token = await ConfigDao().get('tushare_token');
```
