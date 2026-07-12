# 写一个依赖框架的新插件

> 中文说明。English version: [plugin-authoring.en.md](plugin-authoring.en.md)

本文说明如何从零写一个依赖 EM.Framework 的功能插件。假设你已经了解 C# 和 BepInEx IL2CPP 插件的基础。

## 1. 工程引用

功能插件工程要引用 EM.Sdk 与 EM.Framework，但都不拷贝它们的 dll（框架 dll 由 EM.Framework 全局承载一份）。在 csproj 里对这两个引用设 `Private=false`：

```xml
<ItemGroup>
  <ProjectReference Include="..\..\EM.Sdk\EM.Sdk.csproj" Private="false" />
  <ProjectReference Include="..\EM.Framework\EM.Framework.csproj" Private="false" />
</ItemGroup>
```

对 EM.Sdk 的引用是为了用它暴露的钩子与工具；对 EM.Framework 的引用只是为了借 `FrameworkInfo.Guid` 做编译期常量。

目标框架用 `netstandard2.1`，与框架和其它插件一致。

## 2. 声明依赖框架

在插件类上，用 `[BepInDependency]` 显式声明依赖框架。这样 BepInEx 会保证框架先加载，并在框架缺失时明确报出缺失依赖，而不是让插件裸崩。

```csharp
using BepInEx;
using BepInEx.Unity.IL2CPP;

[BepInPlugin(PluginGuid, PluginName, PluginVersion)]
[BepInDependency(EM.Framework.FrameworkInfo.Guid)]
public sealed class MyPlugin : BasePlugin
{
    public const string PluginName = "EM MyPlugin";
    public const string PluginGuid = EM.Sdk.Sdk.PluginGuidPrefix + ".myplugin";
    public const string PluginVersion = EM.Sdk.Sdk.Version;

    public override void Load()
    {
        // 见下一节。
    }
}
```

GUID 用 `EM.Sdk.Sdk.PluginGuidPrefix + ".<模块名>"`的形式，版本用 `EM.Sdk.Sdk.Version`。

## 3. 加载时先做兼容自检

在装任何钩子之前，先过兼容自检门。不通过就直接返回、不装补丁，以保护存档。

```csharp
public override void Load()
{
    var gate = EM.Sdk.CompatibilityGate.Check(Log, PluginName);
    if (!gate.Passed)
    {
        Log.LogWarning($"[{PluginName}] 兼容自检未通过，本插件不安装任何补丁。");
        return;
    }

    var harmony= new HarmonyLib.Harmony(PluginGuid);
    InstallHooks(harmony);

    Log.LogInfo($"[{PluginName}] 已就绪。");
}
```

## 4. 装钩子

优先用 EM.Sdk 已经提供的通用钩子，不要自己另写一份 Harmony。这是本套件确立的开发标准：通用的游戏交互钩子一律上升到 SDK 共享，插件只订阅或调用。各钩子的用法见 [sdk-reference.md](sdk-reference.md)。

```csharp
private void InstallHooks(HarmonyLib.Harmony harmony)
{
    EM.Sdk.Hooks.TournamentHooks.OnTournamentCompleted += OnTournamentCompleted;
    EM.Sdk.Hooks.TournamentHooks.TryInstall(harmony, Log);
}

private void OnTournamentCompleted(EM.Sdk.Hooks.TournamentHooks.TournamentContext ctx)
{
    var tournament = ctx.TournamentInstance as DataTournament;
    if (tournament == null) return;
    // 你的逻辑。
}
```

如果 SDK 里没有你需要的钩子，先判断它是不是通用的：

- 通用的、不依赖游戏 UI 的逻辑/数据类钩子，应当上升到 EM.Sdk，而不是写在插件里各自一份。
- 依赖游戏 UI 程序集的钩子（面板刷新、控件构建等）不进 EM.Sdk，另由独立的 UI 框架承载。
- 确属本插件私有、不具通用性的一次性钩子，才留在插件内，且集中一处安装。

## 5. 几条必须遵守的约定

- 按名解析类型和方法（Harmony `AccessTools`），绝不硬编码地址或偏移。需要的类名、方法名放到 `EM.Core.GameConstants`，那是单一事实源。
- 订阅回调里要能容忍拿到的游戏对象转型失败（返回 null 时安全退出），不要假设一定拿得到。
- 不改游戏的评分数学、模拟、经济等业务逻辑，除非任务明确授权。默认只做加法。
- 默认不写真实存档。若功能需要写存档，必须单独立项、单独批准，满足先备份、可回滚、可重载验证等条件。
- 插件自身的异常不要抛回游戏关键路径。SDK 钩子已经为你隔离了订阅者异常，但你自己额外挂的 Harmony 补丁要自己包好 try/catch。

## 6.构建与部署

- 本地构建需要装了游戏与 BepInEx 的机器，因为要解析游戏目录里的 interop dll。构建配置用 Release。
- 部署到游戏用 `scripts/deploy-plugins.ps1`，它会自动发现含 `[BepInPlugin]` 的插件工程，把框架的共享文件集只放进 `EM.Framework` 目录，各功能插件目录只放自身 dll，并做一致性校验。
- 把开发产物整理进发布目录用 `scripts/sync-readytogo.ps1`，它按白名单挑产物、只取发布必需的 dll、摆进 `ReadyToGo` 下对应结构。

## 7. 自检清单

发布一个新插件前，对照确认：

- 插件类带了 `[BepInDependency(EM.Framework.FrameworkInfo.Guid)]`。
- csproj 对 EM.Sdk 与 EM.Framework 的引用都设了 `Private=false`，发布目录里没有多带三件套。
- `Load()` 里先过 `CompatibilityGate.Check`，不通过就不装补丁。
- 用的是 SDK 已有的通用钩子，没有重复造一份。
- 类名、方法名都在 `GameConstants` 里，没有散落的硬编码字符串或地址。
- 启动游戏看 `LogOutput.log`：框架先加载，本插件随后正常加载，无 `MissingMethodException` / `TypeLoadException`。
