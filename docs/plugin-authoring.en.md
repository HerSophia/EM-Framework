# Writing a New Plugin on the Framework

> English version.中文说明：[plugin-authoring.md](plugin-authoring.md)

This document explains how to write a feature plugin that depends on EM.Framework, from scratch. It assumes you know C# and the basics of BepInEx IL2CPP plugins.

## 1. Project references

A feature plugin project references EM.Sdk and EM.Framework, but copies neither of their dlls (the framework dlls are hosted once, globally, by EM.Framework). Set `Private=false` on both references in the csproj:

```xml
<ItemGroup>
  <ProjectReference Include="..\..\EM.Sdk\EM.Sdk.csproj" Private="false" />
  <ProjectReference Include="..\EM.Framework\EM.Framework.csproj" Private="false" />
</ItemGroup>
```

The EM.Sdk reference is for the hooks and helpersit exposes; the EM.Framework reference is only to use `FrameworkInfo.Guid` as a compile-time constant.

Use `netstandard2.1` as the target framework, consistent with the framework and the other plugins.

## 2. Declare the framework dependency

On the plugin class, declare the framework dependency with `[BepInDependency]`. BepInEx will then load the framework first and, if it is missing, report the missing dependency clearly instead of letting the plugin crash bare.

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
        // see the next section.
    }
}
```

Use `EM.Sdk.Sdk.PluginGuidPrefix + ".<module>"` for the GUID and `EM.Sdk.Sdk.Version` for the version.

## 3. Run the compatibility check first on load

Before installing any hook, pass the compatibility gate. If it fails, return without installing patches, to keep saves safe.

```csharp
public override void Load()
{
    var gate = EM.Sdk.CompatibilityGate.Check(Log, PluginName);
    if (!gate.Passed)
    {
        Log.LogWarning($"[{PluginName}] compatibility check failed; this plugin installs no patch.");
        return;
    }

    var harmony = new HarmonyLib.Harmony(PluginGuid);
    InstallHooks(harmony);

    Log.LogInfo($"[{PluginName}] ready.");
}
```

## 4. Install hooks

Prefer the shared hooks EM.Sdk already provides; do not write your own Harmony patch for them. This is the standard this kit sets: general game-interaction hooks are raised into the shared SDK, and plugins only subscribe or call. See [sdk-reference.en.md](sdk-reference.en.md) for each hook's usage.

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
    // your logic.
}
```

If the SDK does not have the hook you need, first decide whether it is general:

- A general logic/data hook that does not depend on game UI should be raised into EM.Sdk, not written per-plugin.
- A hook that depends on game UI assemblies (panel refresh, control building, and so on)does not go into EM.Sdk; it is carried by a separate UI framework.
- Only a one-off hook that is genuinely private to this plugin stays in the plugin, and is installed in one place.

## 5. Conventions you must follow

- Resolve types and methods by name (Harmony `AccessTools`), never by hardcoded address or offset. Put the type and method names you need into `EM.Core.GameConstants`, the single source of truth.
- In a subscriber callback, tolerate a failed cast of the game object (exit safely when it is null);do not assume you always get it.
- Do not change the game's rating math, simulation, or economy, unless the task explicitly authorizes it. By default, add only.
- Do not write real saves by default. If a feature must write saves, it requires a separate proposal and approval, with backup, rollback, and reload-verification.
- Do not throwyour own exceptions back into a critical gamepath. The SDK hooks already isolate subscriber exceptions for you, but any extra Harmony patch you install yourself must wrap itsown try/catch.

## 6. Build and deploy

- A local build needs a machine with the game and BepInEx installed, because it resolves the interop dlls in the game folder. Build in Release.
- Deploy to the game with `scripts/deploy-plugins.ps1`. It discovers plugin projects with `[BepInPlugin]`, puts the framework's shared file set only into the `EM.Framework` folder, puts only its own dll into each feature plugin folder, and runs a consistency check.
- Stage dev output into the release folder with`scripts/sync-readytogo.ps1`. It picks products by a whitelist, takes only the dlls needed for release, and lays them into the matching structure under `ReadyToGo`.

## 7. Self-check list

Before releasing a new plugin, confirm:

- The plugin class has`[BepInDependency(EM.Framework.FrameworkInfo.Guid)]`.
- The csproj references to EM.Sdk and EM.Framework both set `Private=false`; the release folder does not carry an extra copy of the three core libraries.
- `Load()` passes `CompatibilityGate.Check` first and installs no patch on failure.
- You use the SDK's existing shared hooks and did not build a duplicate.
- Type and method names live in `GameConstants`; there are no scattered hardcoded strings or addresses.
- Start the game and check `LogOutput.log`: the framework loads first, this plugin loads normally afterward, with no `MissingMethodException`/ `TypeLoadException`.
