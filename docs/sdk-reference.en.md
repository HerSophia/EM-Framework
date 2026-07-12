# EM.Sdk Reference

> English version. 中文说明：[sdk-reference.md](sdk-reference.md)

This document describes, item by item, what `EM.Sdk.dll` exposes. All hooks live under the `EM.Sdk.Hooks` namespace. The snippets assume your plugin already references EM.Sdk and holds a Harmony instance `harmony` and a logger `Log`.

Contents:

1. [Framework identity and version constants](#1-framework-identity-and-version-constants)
2. [Compatibility gate: CompatibilityGate](#2-compatibility-gate-compatibilitygate)
3. [Hook overview](#3-hook-overview)
4. [Tournament finish hook: TournamentHooks](#4-tournament-finish-hook-tournamenthooks)
5. [Training day hooks: TrainingHooks / MapTrainingHooks](#5-training-day-hooks-traininghooks--maptraininghooks)
6. [Rating hooks: RatingHooks](#6-rating-hooks-ratinghooks)
7. [Winrate hook: WinrateHooks](#7-winrate-hook-winratehooks)
8. [Player import hook: PlayerImportHooks](#8-player-import-hook-playerimporthooks)
9. [Config overrides](#9-config-overrides)
10. [Localization](#10-localization)
11. [Test base: Testing](#11-test-base-testing)

---

## 1. Framework identity and version constants

- `EM.Sdk.Sdk.Version`: the SDK semantic version, currently `"0.1.0"`.
- `EM.Sdk.Sdk.PluginGuidPrefix`: the GUID prefix for all official mods, `"com.esportsmanager.modkit"`.
- `EM.Framework.FrameworkInfo.Guid`: the framework plugin GUID, i.e. `PluginGuidPrefix + ".framework"`. Feature plugins reference it in `[BepInDependency]`.
- `EM.Framework.FrameworkInfo.Name`: the framework plugin display name, `"EM Framework"`.

Convention: a feature plugin GUID follows `<prefix>.<module>`, for example `com.esportsmanager.modkit.rating21`.

---

## 2. Compatibility gate: CompatibilityGate

`EM.Sdk.CompatibilityGate` consolidates the startup self-check every plugin has to do — read the running Unity version, locate and hash `global-metadata.dat` with SHA-256, and call `EM.Core.Compatibility.Evaluate` — into one place. Call it once in your plugin's `Load()`.

```csharp
var gate = EM.Sdk.CompatibilityGate.Check(Log, PluginName);
if (!gate.Passed)
{
    Log.LogWarning($"[{PluginName}] compatibilitycheck failed; this plugin installs no patch.");
    return;
}
// install hooks only after passing...
```

`Check` returns a `GateResult` readonly struct:

| Member | Meaning |
| ---- | ---- |
| `Passed` | Whether the check passed (the plugin decides whether to install patches). |
| `UnityMatches` | Whether the running Unity version matches the pinned environment. |
| `MetadataChecked` | Whether the metadata hash was actually compared (false if the file could not be located). |
| `MetadataMatches` | Whether the metadata hash matches (false if not compared). |
| `ReportedUnityVersion` | The running Unity version that was read. |
| `ReportedMetadataSha` | The computed metadata SHA-256. |

Policy: a Unity version mismatch is always rejected; a metadata file that is located but whose hash differs is also rejected; if the file cannot be located, the check passes on the Unity version alone and logs a warning. `Check` prints uniform log lines itself; the second argument is a log prefix, usually the plugin name.

---

## 3. Hook overview

| Hook | Attach point | Meaning | Context |
| ---- | ---- | ---- | ---- |
| `TournamentHooks` | `DataTournament.Finish` (Postfix) | A tournament just finished | `TournamentInstance` (native DataTournament) |
| `TrainingHooks` | `TrainingEventEngine.NextDay` (Postfix) | Fitness/attribute training day advanced | `EngineInstance` (native TrainingEventEngine) |
| `MapTrainingHooks` | `MapTrainingEngine.NextDay` (Postfix) | Map training day advanced | `EngineInstance` (native MapTrainingEngine) |
| `RatingHooks` (event) | `MapRecord.GetImpactRating` (Postfix) | Rating computed; reshape the return value | `Original` / `Result` |
| `RatingHooks` (replace) | `MapRecord.GetImpactRating` (Prefix) | Take over rating computation entirely | `RecordInstance` / `Result` |
| `WinrateHooks` | `MatchesEngine.GetTeamOneWinPercentage` (Postfix) | Quick/auto sim winrate computed; overwrite it | `Original` / `Result` / `RawWasPercent` |
| `PlayerImportHooks` | `ImportDataEMDBPlayer.ImportCSVData` (Postfix) | A batch of EMDB players imported | `ImporterInstance` (native ImportDataEMDBPlayer) |

Conventions shared by all hooks:

- `TryInstall(Harmony, ManualLogSource?)` is idempotent: repeated installs succeed safely; if the target type/method is not found it returns false rather than throwing.
- Game objects in a context are carried as `object`; the subscriber casts them (safe cast or reflection).
- Subscriber exceptions are isolated one by one (a try/catch per subscriber), so they never break a critical game path.

---

## 4. Tournament finish hook: TournamentHooks

Fires after `DataTournament.Finish()` returns. The Postfix `__instance` is the tournament that just finished.

```csharp
using EM.Sdk.Hooks;

TournamentHooks.OnTournamentCompleted += ctx =>
{
    // ctx.TournamentInstance is the native DataTournament; cast or reflect yourself.
    var tournament = ctx.TournamentInstance as DataTournament;
    if (tournament ==null) return;
// do your work after the tournament ends, e.g. read placements, grant rewards...
};

TournamentHooks.TryInstall(harmony, Log);
```

- Event: `event Action<TournamentContext> OnTournamentCompleted`
- Context: `TournamentContext.TournamentInstance` (`object`, native DataTournament)

---

## 5. Training day hooks: TrainingHooks / MapTrainingHooks

These are two different engines and two different daily paths; pick one or both:

- `TrainingHooks` attaches to `TrainingEventEngine.NextDay`, for fitness/attribute training (physio, psychology, and so on).
- `MapTrainingHooks` attaches to `MapTrainingEngine.NextDay`, for map training.

The interfaces are identical. The Postfix timing guarantees the game has settled thecurrent day's training, so a subscriber can plan later training without affecting the current day.

```csharp
using EM.Sdk.Hooks;

TrainingHooks.OnDayAdvanced += ctx =>
{
    var engine = ctx.EngineInstance as TrainingEventEngine;
    if (engine == null) return;
    // the current day's training is settled; read or schedule what comes next.
};

TrainingHooks.TryInstall(harmony, Log);
```

- Event: `event Action<TrainingContext> OnDayAdvanced` (MapTrainingHooks uses `Action<MapTrainingContext>`)
- Context: `EngineInstance` (`object`, the corresponding engine instance)

---

## 6. Rating hooks: RatingHooks

`RatingHooks` offers two paths on `MapRecord.GetImpactRating`; choose one.

### 6.1 Event reshape (Postfix)

Take the game's computed rating and reshape the return value on top of it. Use this when you want to recalibrate the original value.

```csharp
using EM.Sdk.Hooks;

RatingHooks.OnImpactRatingComputed += ctx =>
{
    // ctx.Original is the game's raw return; set ctx.Result to change what the game returns.
    ctx.Result = Recalibrate(ctx.Original);
};

RatingHooks.TryInstall(harmony, Log);
```

- Event: `event Action<ImpactRatingContext> OnImpactRatingComputed`
- Context: `ImpactRatingContext.Original` (raw value), `ImpactRatingContext.Result` (set this)

### 6.2 Engine replacement (Prefix)

Take over rating computation entirely, replacing the game's implementation with yourown engine. Use this when you must replace rather than reshape (the game's baseline is a compile-time inlined constant that cannot be changed via a field, so only full replacement works).

```csharp
using EM.Sdk.Hooks;

RatingHooks.InstallReplacer(harmony, ctx =>
{
    var record = ctx.RecordInstance as MapRecord;
    if (record == null) return false;   // return false: let the original run (graceful fallback).
    ctx.Result = MyEngine.Compute(record);
    return true;                        // return true: use ctx.Result and skip the original.
}, Log);
```

- Method: `bool InstallReplacer(Harmony, Func<RatingReplaceContext, bool> replacer, ManualLogSource?)`
- Context: `RatingReplaceContext.RecordInstance` (`object`, native MapRecord), `RatingReplaceContext.Result` (replacement value)
- Returning `true` means you provided a replacement and skip the original; `false` lets the original run. Any exception inside the callback is swallowed and the original is allowedto run, so the game never crashes.

Note: the event path and the replacement path can coexist, but once a replacement callback returns true and skips the original, registered Postfix events still fire on the replaced result. On the same `GetImpactRating`, two plugins that both want full replacement conflict; use only one.

---

## 7. Winrate hook: WinrateHooks

In quick/auto simulation, fires after `MatchesEngine.GetTeamOneWinPercentage` returns; lets you overwrite Team1's win probability.

```csharp
using EM.Sdk.Hooks;

WinrateHooks.OnWinPercentageComputed += ctx =>
{
    // ctx.Original is normalized to [0,1]. Set ctx.Result (also in [0,1]).
    ctx.Result = Reshape(ctx.Original);
};

WinrateHooks.TryInstall(harmony, Log);
```

- Event: `event Action<WinPercentageContext> OnWinPercentageComputed`
- Context: `Original` (raw probability, `[0,1]`), `Result` (set this, `[0,1]`), `RawWasPercent` (diagnostic: whether the raw return was on a percentage scale)
- The SDK restores the original scale (if the game returned a percentage, the SDK multiplies back).

One caveat: the value here has already been clamped by the game to roughly `[0.2, 0.8]`, so information is lost. To recompute a blowout from the two teams' power difference, read the two teams' power directly with strongly typed interop inside your plugin, not in this game-free SDK layer.

---

## 8. Player import hook: PlayerImportHooks

Fires after a batch of EMDB players finish importing in `ImportDataEMDBPlayer.ImportCSVData()`, at which point the batch is fully written into the global `DataPlayers` collection.

```csharp
using EM.Sdk.Hooks;

PlayerImportHooks.OnPlayersImported += ctx =>
{
    // Player data is in the global DataPlayers collection; iterate DataPlayers.List.
    // ctx.ImporterInstance is the native ImportDataEMDBPlayer that triggered this import.
    foreach (var p in DataPlayers.List) { /* handle matched players */ }
};

PlayerImportHooks.TryInstall(harmony, Log);
```

- Event: `event Action<PlayerImportContext> OnPlayersImported`
- Context: `PlayerImportContext.ImporterInstance` (`object`, native ImportDataEMDBPlayer)

Only the EMDB variant of `ImportCSVData` is hooked: it is a plain synchronous method that has completed the import when it returns. The non-EMDB `ImportDataPlayer.ImportCSVData` is an async state machine whose Postfix returns when the coroutine starts, which is the wrong moment, so it is not hooked. The in-game Database → Upload EMDB flow uses the EMDB variant.

---

## 9. Config overrides

`EM.Sdk.Config` is the L1 override engine: it loads `em-overrides/*.json` from a directory, validates them, dispatches to registered appliers, and supports hot reload. It suits config-style mods that change behavior without code.

```csharp
using EM.Sdk.Config;

var engine = new OverrideEngine();
engine.Register(new BalanceApplier(harmony));    // rating section
engine.Register(new BuyConfigApplier(harmony));  // buy section
engine.Initialize(Path.Combine(Paths.ConfigPath, "em-overrides"), Log);

// Hot reload (e.g. on a dev-bridge reload request):
engine.Reload(Log);
```

- `OverrideEngine`: `Register(IOverrideApplier)`, `Initialize(dir, log)`, `Reload(log)`, `Store` (the underlying store with the merged document and issue list).
- `IOverrideApplier`: the interface for one section's applier. `Section` is the top-level sectionname (such as `buy` / `rating`); `Apply(doc, log)` applies the validated document to the game. `Apply` must be idempotent: the overlay is the single source of truth, repeated calls give a stable result, no incremental accumulation.
- Built-in appliers:
  - `BalanceApplier(harmony)`: the `rating` section. Turns config into a rating recalibration curve and reuses the `RatingHooks` Postfix to reshape the return value.
  - `BuyConfigApplier(harmony)`: the `buy` section. Prefixes `SimulationMod.SetupBuyQueue` to rewrite the buy config.
- `OverrideStore`: pure IO plus delegation to `EM.Core.Config` for parse/merge/validate; a dev bridge or CLI can callit directly to read `Issues`.

Validation rule: if a section has an error, that section's applier is skipped (other sections proceed); a syntax or root-level error skips all application. All issues are exposed through `Store`.

---

## 10. Localization

`EM.Sdk.Localization` provides language-pack injection. The pure parse/merge/lookup logic is in `EM.Core.Localization` (offline testable); this layer only triggers IO, caches state, and talks to the game.

- `LocalizationOverlay`: the plugin-side holder and facade for language packs.
  - `LoadFrom(dir, log)`: loads packs from a directory and rebuilds the catalog; thread-safe and repeatable (hot reload). Default directory name `em-locale` (under `BepInEx/config/`).
  - `Catalog` / `Issues` / `TryTranslate(localeCode, table, key, out value)` / `HasLocale(localeCode)`.
- `LocalizationInjector`: the main-path injection. Writes a locale's entries into the game's current Unity `StringTable` (overwrite existing keys, add missing ones), through the game's native lookup path so placeholders work naturally.
- `IsLanguageReady(log)`: whether the game's localization has finished async initialization.
  - `Inject(localeCode, log)`: injects that language pack into the currently selected locale's tables and returns the number of entries written. Must be called on the main thread.
- `LocalizationFunnel`: a lookup funnel, dev / opt-in, not installed by default. Two uses: one, raise each lookup's `(table, key, return value)` through the `Observed` event to a subscriber (to scan for missing keys); two, when `ReplaceReturns = true`, replace return values as a fallback for lazily loaded entries that injection missed. The downside is the funnel receives an already-formatted string, so a replacement no longer has placeholder arguments; it is a fallback only, and the main path always prefers `LocalizationInjector`.

```csharp
using EM.Sdk.Localization;

LocalizationOverlay.LoadFrom(Path.Combine(Paths.ConfigPath, "em-locale"), Log);
if (LocalizationInjector.IsLanguageReady(Log))
{
    int n = LocalizationInjector.Inject("zh-Hans", Log);
    Log.LogInfo($"injected {n} entries.");
}
```

---

## 11.Test base: Testing

`EM.Sdk.Testing` is the infrastructure for the in-game automation runner (the B layer). It is for development only and never ships with the user build.

- `TestRunnerPluginBase`: an abstract base that inherits `BasePlugin`. A subclass implements `RunTests()`: install hooks / build samples → call game methods → assert → collect results.
  - `Harmony` / `Results` / `CompatibleGame`.
  - `Finish(goldenSamples?)`: writes results to disk (default `BepInEx/em-test-results.json`) and requests game exit, optionally with a golden manifest.
  - Most cases use passive capture: install Harmony hooks in `RunTests()`,and let the real assertions happen inthe callbackswhen the game triggers naturally (on the main thread), which satisfies the constraint that il2cpp access must be on the main thread.
- `TestAssert`: a lightweight assertion helper; each assertion produces one `TestResult` (no exceptions, so they can be collected in bulk and written to disk).
  - `Approximately(name, actual, expected, tolerance, extra)`
  - `InRange(name, actual, lo, hi, extra)`
  - `True(name, condition, message)`
- `TestResult` / `TestResults` / `TestResultsWriter`: the result model and disk writer.
