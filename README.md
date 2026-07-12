# EM.Framework — Shared Framework for Esports Manager 2026 Mods / Mod 共享框架

> Version 0.1.1 ｜ Target game / 适配游戏：Esports Manager 2026 Steam retail patch 1.0.2 (Unity 6000.3.12f1 / IL2CPP metadata v39) ｜ License: MIT

**Language / 语言**: [English](#english) ｜ [中文](#中文)

---

<a name="english"></a>

## English

### Table of Contents

- [What is in this folder](#what-is-in-this-folder)
- [Why a shared framework](#why-a-shared-framework)
- [How to install](#how-to-install)
- [Version andcompatibility](#version-and-compatibility)
- [Further reading](#further-reading)
- [Conventions and boundaries](#conventions-and-boundaries)

EM.Framework is the shared runtime foundation used by every mod built on this kit. It gives the three core libraries (EM.Core / EM.Data / EM.Sdk) a host with a BepInEx plugin identity, so each feature plugin depends on it and shares one copy of the framework inside the game, instead of each carrying its own copy.

### What is in this folder

| File | Purpose |
| ---- | ---- |
| `EM.Framework.dll` | The framework host plugin. Has a `[BepInPlugin]` identity, prints one version line on load, installs no hooks itself. |
| `EM.Sdk.dll` |The shared foundation. Exposes hooks, the compatibility gate, config overrides, localization, and a test base class. |
| `EM.Core.dll` | Pure logic layer. Rating baseline, compatibility checks, game constants (single source of truth for type/method names), with zero Unity dependency. |
| `EM.Data.dll` | Data format layer. EMDB pack/unpack, player CSV schema, and so on. |
| `BouncyCastle.Cryptography.dll` | A transitive dependency of EM.Data, used for AES-GCM. |

These five files together form the complete framework. A feature plugin folder holds only that plugin's own dll and never repeats any of the files above.

### Why a shared framework

Earlier, every plugin folder carried its own copy of the three core libraries, which caused recurring failures: in the BepInEx runtime, an assembly with a given name is loaded only once — the first copy encountered (by plugin folder name order). An older copy carried by an alphabetically earlier plugin would win, so a later plugin calling a new method or loading a new type would throw `MissingMethodException` / `TypeLoadException`.

EM.Framework removes this by design: the three libraries ship only with this framework, so there is exactly one copy in the game. Each feature plugin declares its dependency explicitly with `[BepInDependency(EM.Framework.FrameworkInfo.Guid)]`. Same-name assembly shadowing canno longer happen, and load order isguaranteed.

### How to install

1. Make sure BepInEx (IL2CPP, BE#785) is installed and the game starts normally.
2. Copy this whole folder (`EM.Framework/`) into the game's `BepInEx/plugins/`.
3. Copy the feature plugin folders you want into `BepInEx/plugins/` as well. Each feature plugin folder contains only its own dll.
4. Start the game and check `BepInEx/LogOutput.log`: EM.Framework should print its version line first, then the feature plugins load normally.

Layout (under `BepInEx/plugins/`):

```text
plugins/
  EM.Framework/        this framework (five dlls, one copy for the whole game)
  Rating21/         a feature plugin (only Rating21.dll)
  AutoTraining/        a feature plugin (only AutoTraining.dll)
  ...
```

### Version and compatibility

- Framework version: 0.1.0 (`EM.Sdk.Sdk.Version`).
- Target game: Unity 6000.3.12f1, IL2CPP metadata v39.
- BepInEx: 6.0.0-be.785 (IL2CPP-win-x64). The official release does not support metadata v39; BE#785 is required.

The game receives patches over time.Each feature plugin runs a compatibility self-check on load (a dual gate on the Unity version and the `global-metadata.dat` hash). On a mismatch the plugin degrades to warning only and applies no patch, to keep saves safe from half-compatible logic. See the compatibility gate section in [docs/sdk-reference.en.md](docs/sdk-reference.en.md).

### Further reading

- [docs/sdk-reference.en.md](docs/sdk-reference.en.md): a per-item reference of everything EM.Sdk exposes (hooks, compatibility gate, config overrides, localization, test base), eachwith a minimal usable code snippet.
- [docs/plugin-authoring.en.md](docs/plugin-authoring.en.md): the full flow for writing a new plugin that depends on this framework.

### Conventions and boundaries

- The framework changes no game business logic; it only provides entry points. Rating math, simulation, and economy are left untouched.
- Alltypes and methods are resolved by name (Harmony `AccessTools`), never by hardcoded address or offset. Type and method names live in `EM.Core.GameConstants` as the single source of truth.
- Game objects in a hook context are always carried as `object`; the subscriber casts them. This keeps EM.Sdk free of compile-time references to game interop assemblies and resilient to game updates.
- Hooks that touch game UI do not live in EM.Sdk; they are carried by a separate UI framework (planned). EM.Sdk holds only general logic and data hooks.

---

<a name="中文"></a>

## 中文

### 目录

- [这个目录里有什么](#这个目录里有什么)
- [为什么需要这样一份共享框架](#为什么需要这样一份共享框架)
- [怎么安装](#怎么安装)
- [版本与兼容](#版本与兼容)
- [进一步阅读](#进一步阅读)
- [约定与边界](#约定与边界)

EM.Framework 是所有基于本套件开发的 mod 共用的运行时地基。它给三件套（EM.Core / EM.Data / EM.Sdk）一个有 BepInEx 插件身份的宿主，让每个功能插件都依赖它、在游戏里共用同一份框架，而不再各自带一份。

### 这个目录里有什么

| 文件 | 作用 |
| ---- | ---- |
| `EM.Framework.dll` | 框架承载插件。带 `[BepInPlugin]` 身份，加载时打一行版本日志，本身不装任何钩子。 |
| `EM.Sdk.dll` | 共享地基。对外暴露钩子、兼容自检、配置覆盖、本地化、测试基类等能力。 |
| `EM.Core.dll` | 纯逻辑层。评分基线、兼容判定、游戏常量（类名/方法名单一事实源）等，零 Unity 依赖。 |
| `EM.Data.dll` | 数据格式层。EMDB 打解包、选手 CSV schema等。 |
| `BouncyCastle.Cryptography.dll` | EM.Data 的传递依赖，供 AES-GCM 使用。 |

这五个文件合起来就是一份完整的框架。功能插件目录里只放插件自己的 dll，不再重复携带以上任何一个。

### 为什么需要这样一份共享框架

早期每个插件目录各带一份三件套，反复出问题：BepInEx 的运行时里，同名程序集只加载最先遇到的那一份（按插件目录字母序）。字母序靠前的插件带的旧三件套会抢占，导致后加载的插件调用新方法或加载新类型时抛 `MissingMethodException` / `TypeLoadException`。

EM.Framework 从结构上解决它：三件套只随本框架部署，游戏里全局只有一份；各功能插件通过 `[BepInDependency(EM.Framework.FrameworkInfo.Guid)]` 显式声明依赖它。同名程序集抢占从此不可能再发生，加载顺序也有了保证。

### 怎么安装

1. 确认已装好 BepInEx（IL2CPP，BE#785），且游戏可正常启动。
2. 把本目录（`EM.Framework/`）整体拷进游戏的 `BepInEx/plugins/`。
3. 把要用的各功能插件目录也拷进 `BepInEx/plugins/`。每个功能插件目录只含它自己的 dll。
4. 启动游戏，查看 `BepInEx/LogOutput.log`：应先看到 EM.Framework 打出版本日志，各功能插件随后正常加载。

目录结构示意（`BepInEx/plugins/` 下）：

```text
plugins/
  EM.Framework/     本框架（五个 dll，全局一份）
  Rating21/            某功能插件（只含 Rating21.dll）
  AutoTraining/        某功能插件（只含 AutoTraining.dll）
  ...
```

### 版本与兼容

- 框架版本：0.1.0（`EM.Sdk.Sdk.Version`）。
- 目标游戏：Unity 6000.3.12f1，IL2CPP metadata v39。
- BepInEx：6.0.0-be.785（IL2CPP-win-x64）。官方 release不支持 metadata v39，必须用 BE#785。

游戏是会打补丁的。每个功能插件在加载时会做兼容自检（Unity 版本 + `global-metadata.dat` 哈希双门禁）。不匹配时插件降级为只警告、不打补丁，以保护存档不被半兼容逻辑污染。详见 [docs/sdk-reference.md](docs/sdk-reference.md) 的兼容自检一节。

### 进一步阅读

- [docs/sdk-reference.md](docs/sdk-reference.md)：EM.Sdk 对外暴露的全部能力逐项参考（钩子、兼容自检、配置覆盖、本地化、测试基类），每项附最小可用代码。
- [docs/plugin-authoring.md](docs/plugin-authoring.md)：从零写一个依赖本框架的新插件的完整流程。

### 约定与边界

- 框架不改任何游戏业务逻辑，只提供接入点。评分数学、模拟、经济等一律不碰。
- 所有类型/方法一律按名解析（Harmony `AccessTools`），绝不硬编码地址或偏移。类名、方法名集中在 `EM.Core.GameConstants`，是单一事实源。
- 钩子上下文里的游戏对象一律用 `object` 承载，订阅方自行转型。这样 EM.Sdk 不在编译期引用游戏 interop 程序集，抗游戏更新。
-涉及游戏 UI 的钩子不放在 EM.Sdk，另由独立的 UI 框架承载（规划中）。EM.Sdk 只收纯逻辑与数据类的通用钩子。
