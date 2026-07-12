# EM.Sdk 参考

> 中文说明。English version: [sdk-reference.en.md](sdk-reference.en.md)

本文逐项说明 `EM.Sdk.dll` 对外暴露的能力。所有钩子都在 `EM.Sdk.Hooks` 命名空间下。示例假设你的插件已经引用 EM.Sdk、并持有一个 Harmony 实例 `harmony` 与一个日志 `Log`。

目录：

1. [框架身份与版本常量](#1-框架身份与版本常量)
2. [兼容自检门 CompatibilityGate](#2-兼容自检门-compatibilitygate)
3. [钩子总览](#3-钩子总览)
4. [赛事结算钩子 TournamentHooks](#4-赛事结算钩子-tournamenthooks)
5. [训练日推进钩子 TrainingHooks / MapTrainingHooks](#5-训练日推进钩子-traininghooks--maptraininghooks)
6. [评分钩子 RatingHooks](#6-评分钩子-ratinghooks)
7. [胜率钩子 WinrateHooks](#7-胜率钩子-winratehooks)
8. [选手导入钩子 PlayerImportHooks](#8-选手导入钩子-playerimporthooks)
9. [配置覆盖 Config](#9-配置覆盖-config)
10. [本地化 Localization](#10-本地化-localization)
11. [测试基类 Testing](#11-测试基类-testing)

---

## 1. 框架身份与版本常量

- `EM.Sdk.Sdk.Version`：SDK 语义化版本，当前 `"0.1.0"`。
- `EM.Sdk.Sdk.PluginGuidPrefix`：所有官方 mod 的 GUID 前缀，`"com.esportsmanager.modkit"`。
- `EM.Framework.FrameworkInfo.Guid`：框架插件 GUID，即 `PluginGuidPrefix + ".framework"`。功能插件在 `[BepInDependency]`里引用它。
- `EM.Framework.FrameworkInfo.Name`：框架插件显示名，`"EM Framework"`。

约定：功能插件的 GUID用 `<前缀>.<模块名>` 的形式，例如 `com.esportsmanager.modkit.rating21`。

---

## 2. 兼容自检门 CompatibilityGate

`EM.Sdk.CompatibilityGate` 把「读运行中 Unity 版本 +定位并算 `global-metadata.dat` 的 SHA-256 + 调 `EM.Core.Compatibility.Evaluate`」这套每个插件都要做的启动自检收敛为一处。在插件的 `Load()` 里调一行即可。

```csharp
var gate = EM.Sdk.CompatibilityGate.Check(Log, PluginName);
if (!gate.Passed)
{
    Log.LogWarning($"[{PluginName}] 兼容自检未通过，本插件不安装任何补丁。");
    return;
}
// 通过后再安装钩子……
```

`Check` 返回一个 `GateResult` 只读结构：

| 成员 | 含义 |
| ---- | ---- |
| `Passed` | 是否通过（插件据此决定是否安装补丁）。 |
| `UnityMatches` | 运行中 Unity 版本是否匹配已固化环境。 |
| `MetadataChecked` | 本次是否实际比对了 metadata 哈希（文件定位不到时为 false）。 |
| `MetadataMatches` | metadata 哈希是否匹配（未比对时为 false）。 |
| `ReportedUnityVersion` | 读到的运行中 Unity 版本。 |
| `ReportedMetadataSha` | 算出的 metadata SHA-256。 |

判定策略：Unity 版本不匹配一律拒绝；metadata 文件能定位但哈希不符也拒绝；文件定位不到则仅凭 Unity 版本放行并记一条警告。`Check` 自己会打印统一格式的日志，第二个参数是日志前缀，一般传插件名。

---

## 3. 钩子总览

| 钩子 | 挂载点 | 语义 | 上下文 |
| ---- | ---- | ---- | ---- |
| `TournamentHooks` | `DataTournament.Finish` (Postfix) | 一个赛事刚结算完成 | `TournamentInstance`（原生 DataTournament） |
| `TrainingHooks` | `TrainingEventEngine.NextDay` (Postfix) | 体能/属性训练日推进结束 | `EngineInstance`（原生 TrainingEventEngine） |
| `MapTrainingHooks` | `MapTrainingEngine.NextDay` (Postfix) | 地图训练日推进结束 | `EngineInstance`（原生 MapTrainingEngine） |
| `RatingHooks`（事件） | `MapRecord.GetImpactRating` (Postfix) | 赛后 rating 计算完，可重塑返回值 | `Original` / `Result` |
| `RatingHooks`（替换） | `MapRecord.GetImpactRating` (Prefix) | 整体接管 rating 计算 | `RecordInstance` / `Result` |
| `WinrateHooks` | `MatchesEngine.GetTeamOneWinPercentage` (Postfix) | 快速/自动模拟胜率算完，可改写 | `Original` / `Result` / `RawWasPercent` |
| `PlayerImportHooks` | `ImportDataEMDBPlayer.ImportCSVData` (Postfix) | 一批 EMDB 选手导入完成 | `ImporterInstance`（原生 ImportDataEMDBPlayer） |

所有钩子共有的约定：

- 安装方法 `TryInstall(Harmony, ManualLogSource?)` 是幂等的：重复安装安全成功；找不到目标类型/方法时返回 false 而不抛异常。
- 上下文里的游戏对象用 `object` 承载，订阅方自行转型（安全转型或反射）。
- 订阅者的异常被逐个隔离（每个订阅者 try/catch），不会反向打断游戏关键路径。

---

## 4. 赛事结算钩子 TournamentHooks

一个赛事在 `DataTournament.Finish()` 返回后触发。Postfix 的 `__instance` 就是刚结算完成的赛事本体。

```csharp
using EM.Sdk.Hooks;

TournamentHooks.OnTournamentCompleted += ctx =>
{
    // ctx.TournamentInstance 是原生 DataTournament，自行转型或反射读取。
    var tournament = ctx.TournamentInstance as DataTournament;
    if (tournament == null) return;
    // 在赛事结束后做自己的事，例如读取名次、发奖励……
};

TournamentHooks.TryInstall(harmony, Log);
```

- 事件：`event Action<TournamentContext> OnTournamentCompleted`
- 上下文：`TournamentContext.TournamentInstance`（`object`，原生 DataTournament）

---

## 5. 训练日推进钩子 TrainingHooks / MapTrainingHooks

这是两个不同的引擎、不同的每日结算路径，按需要选一个或都用：

- `TrainingHooks` 挂 `TrainingEventEngine.NextDay`，对应体能/属性训练（理疗、心理等）。
- `MapTrainingHooks` 挂 `MapTrainingEngine.NextDay`，对应地图训练。

两者接口一致。Postfix 的时机保证原版当天的训练已经结算，订阅者可在随后为后续训练排计划，不影响当天结算。

```csharp
using EM.Sdk.Hooks;

TrainingHooks.OnDayAdvanced += ctx =>
{
    var engine = ctx.EngineInstance as TrainingEventEngine;
    if (engine == null)return;
    // 当天训练已结算，可在此读取或安排后续。
};

TrainingHooks.TryInstall(harmony, Log);
```

-事件：`event Action<TrainingContext> OnDayAdvanced`（MapTrainingHooks 为 `Action<MapTrainingContext>`）

- 上下文：`EngineInstance`（`object`，对应引擎实例）

---

## 6. 评分钩子 RatingHooks

`RatingHooks` 针对 `MapRecord.GetImpactRating` 提供两条路径，按需求择一。

### 6.1 事件式重塑（Postfix）

拿到游戏算出的原始 rating，在其基础上重塑返回值。适合「在原值上做重标定」的场景。

```csharp
using EM.Sdk.Hooks;

RatingHooks.OnImpactRatingComputed += ctx =>
{
    // ctx.Original 是游戏原始返回值；改写 ctx.Result 即改写游戏返回值。
    ctx.Result = Recalibrate(ctx.Original);
};

RatingHooks.TryInstall(harmony, Log);
```

- 事件：`event Action<ImpactRatingContext> OnImpactRatingComputed`
- 上下文：`ImpactRatingContext.Original`（原始值）、`ImpactRatingContext.Result`（改写它）

### 6.2 引擎替换式（Prefix）

整体接管 rating 计算，用自己的引擎重算取代游戏原实现。适合「不在原值上重塑、而要整体替换」的场景（游戏原公式基线是编译期内联常量，改不了字段，只能整体替换）。

```csharp
using EM.Sdk.Hooks;

RatingHooks.InstallReplacer(harmony, ctx =>
{
    var record = ctx.RecordInstance as MapRecord;
    if (record == null) return false;   // 返回 false：放行原实现（优雅降级）。
    ctx.Result = MyEngine.Compute(record);
    return true;                        // 返回 true：用 ctx.Result 替换、跳过原实现。
}, Log);
```

- 方法：`bool InstallReplacer(Harmony, Func<RatingReplaceContext, bool> replacer, ManualLogSource?)`
- 上下文：`RatingReplaceContext.RecordInstance`（`object`，原生 MapRecord）、`RatingReplaceContext.Result`（替换值）
- 回调返回 `true` 表示已给出替换值、跳过原实现；返回 `false` 表示放行原实现。回调内任何异常都会被吞下并放行原实现，绝不让游戏崩。

注意：事件式与替换式可以并存，但当替换回调返回 true 跳过原实现后，已注册的 Postfix 事件仍会在替换结果上触发。同一个 `GetImpactRating` 上，两个插件若都想整体替换会相互冲突，应择一使用。

---

## 7. 胜率钩子 WinrateHooks

快速/自动模拟里，`MatchesEngine.GetTeamOneWinPercentage`返回后触发，可改写 Team1 的获胜概率。

```csharp
using EM.Sdk.Hooks;

WinrateHooks.OnWinPercentageComputed += ctx =>
{
    // ctx.Original 已归一到 [0,1]。改写 ctx.Result（同样以 [0,1] 为口径）。
    ctx.Result = Reshape(ctx.Original);
};

WinrateHooks.TryInstall(harmony, Log);
```

- 事件：`event Action<WinPercentageContext> OnWinPercentageComputed`
- 上下文：`Original`（原始概率，`[0,1]`）、`Result`（改写它，`[0,1]`）、`RawWasPercent`（诊断：原始返回值是否为百分数尺度）
- SDK 负责按原尺度还原（游戏若返回百分数，SDK 会自动乘回）。

一点提醒：这个方法拿到的是已被游戏硬夹到约 `[0.2, 0.8]` 的值，信息已经有损。若要按两队实力差重算还原碾压，需要在插件内用强类型 interop 直接读两队实力，而不是在这个 game-free 的 SDK 层做。

---

## 8. 选手导入钩子 PlayerImportHooks

一批 EMDB 选手在 `ImportDataEMDBPlayer.ImportCSVData()` 返回后触发，此时该批选手已全部写入全局 `DataPlayers` 集合。

```csharp
using EM.Sdk.Hooks;

PlayerImportHooks.OnPlayersImported += ctx =>
{
// 选手数据在全局DataPlayers 集合里，遍历 DataPlayers.List 即可。
    // ctx.ImporterInstance 是触发本次导入的原生 ImportDataEMDBPlayer。
    foreach (var p in DataPlayers.List) { /* 对命中选手做处理 */ }
};

PlayerImportHooks.TryInstall(harmony, Log);
```

- 事件：`event Action<PlayerImportContext> OnPlayersImported`
- 上下文：`PlayerImportContext.ImporterInstance`（`object`，原生 ImportDataEMDBPlayer）

只挂 EMDB 版本的 `ImportCSVData`：它是普通同步方法，返回时导入已完成。非 EMDB 的 `ImportDataPlayer.ImportCSVData` 是异步状态机，Postfix 会在协程启动时就返回，时机不对，故不挂。游戏内 Database → Upload EMDB流程走的正是 EMDB 版本。

---

## 9. 配置覆盖 Config

`EM.Sdk.Config` 是 L1 覆盖引擎：从磁盘目录加载 `em-overrides/*.json`，校验后分发给注册的应用者（applier），支持热重载。适合「零代码改行为」的配置类 mod。

```csharp
using EM.Sdk.Config;

var engine = new OverrideEngine();
engine.Register(new BalanceApplier(harmony));    // rating段
engine.Register(new BuyConfigApplier(harmony));  // buy 段
engine.Initialize(Path.Combine(Paths.ConfigPath, "em-overrides"), Log);

// 热重载（例如收到 dev 桥的重载请求时）：
engine.Reload(Log);
```

- `OverrideEngine`：`Register(IOverrideApplier)`、`Initialize(dir, log)`、`Reload(log)`、`Store`（底层存储，含合并文档与问题列表）。
- `IOverrideApplier`：一个「段」的应用者接口。`Section` 是顶层段名（如 `buy` / `rating`），`Apply(doc, log)` 把校验通过的文档落到游戏上。约定 `Apply` 必须幂等：以 overlay 为唯一真值，重复调用结果稳定，不做增量累加。
- 内置应用者：
  - `BalanceApplier(harmony)`：`rating`段。把配置转成评分重标定曲线，复用 `RatingHooks` 的 Postfix 改写返回值。
  - `BuyConfigApplier(harmony)`：`buy` 段。Prefix `SimulationMod.SetupBuyQueue`，按配置改写买枪配置。
- `OverrideStore`：纯 IO + 委托 `EM.Core.Config` 做解析/合并/校验，可被 dev 桥或 CLI 直接调用读 `Issues`。

校验规则：某段存在 error 则跳过该段的 applier（其它段照常）；语法或根级 error 则跳过全部应用。所有问题通过 `Store` 暴露。

---

## 10.本地化 Localization

`EM.Sdk.Localization` 提供语言包注入能力。纯解析、合并、查找逻辑在 `EM.Core.Localization`（离线可测），本层只做 IO 触发、状态缓存与游戏交互。

- `LocalizationOverlay`：语言包在插件侧的持有点与门面。
  - `LoadFrom(dir, log)`：从目录加载语言包并重建目录，线程安全、可重复调用（热重载）。默认目录名 `em-locale`（位于 `BepInEx/config/` 下）。
  - `Catalog` / `Issues` / `TryTranslate(localeCode, table, key, out value)` / `HasLocale(localeCode)`。
- `LocalizationInjector`：主路径注入。把某个 locale 的条目写进游戏当前的 Unity `StringTable`（已有 key覆写、无 key 新增），走游戏原生取值路径，占位符自然生效。
  - `IsLanguageReady(log)`：游戏本地化是否已异步初始化就绪。
  - `Inject(localeCode, log)`：把该语言包注入到当前选中 Locale 的表，返回成功写入的条目数。必须在主线程调用。
- `LocalizationFunnel`：取值漏斗，dev / opt-in，默认不安装。两种用途：一是把每次取值请求的 `(table, key, 返回值)` 通过 `Observed` 事件抛给订阅方（用于扫描缺失 key）；二是当 `ReplaceReturns = true` 时，对注入不到的懒加载条目做兜底替换。缺点是漏斗拿到的是已格式化字符串，替换后不再有占位符参数，故仅作兜底，主路径始终优先 `LocalizationInjector`。

```csharp
using EM.Sdk.Localization;

LocalizationOverlay.LoadFrom(Path.Combine(Paths.ConfigPath, "em-locale"), Log);
if (LocalizationInjector.IsLanguageReady(Log))
{
    int n = LocalizationInjector.Inject("zh-Hans", Log);
    Log.LogInfo($"注入 {n} 条。");
}
```

---

## 11. 测试基类 Testing

`EM.Sdk.Testing` 是 B 层游戏内自动化跑器的基础设施，仅供开发使用，绝不随用户版分发。

- `TestRunnerPluginBase`：抽象基类，继承 `BasePlugin`。子类实现 `RunTests()`：安装钩子 / 构造样本 → 调游戏方法 → 断言 → 收集结果。
  - `Harmony` / `Results` / `CompatibleGame`。
  - `Finish(goldenSamples?)`：把结果写盘（默认 `BepInEx/em-test-results.json`）并请求退出游戏，可附黄金清单。
  - 多数用例采用被动捕获：在 `RunTests()` 里装 Harmony 钩子，真正的断言发生在游戏自然触发（主线程）时的回调里，天然满足 il2cpp 访问必须在主线程的约束。
- `TestAssert`：轻量断言助手，每个断言产出一条 `TestResult`（不抛异常，便于批量收集后写盘）。
  - `Approximately(name, actual, expected, tolerance, extra)`
  - `InRange(name, actual, lo, hi, extra)`
  - `True(name, condition, message)`
- `TestResult` / `TestResults` / `TestResultsWriter`：结果模型与写盘。
