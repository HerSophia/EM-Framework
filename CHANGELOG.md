# CHANGELOG — EM.Framework

版本号遵循语义化版本。适配游戏：Esports Manager 2026 Steam 正版补丁 1.0.2（Unity 6000.3.12f1 / IL2CPP metadata v39）。

## 0.1.1（2026-07-12）

- 适配 Steam 正版补丁 1.0.2：更新兼容自检所用的 global-metadata.dat 与 GameAssembly.dll 指纹为 1.0.2 的值，同时保留 1.0.1 与 GoldBerg 指纹于已接受列表，命中任一即通过。
- 框架自身逻辑未变，只是随游戏版本更新校验基准。

## 0.1.0（2026-07-12）

- 首个对外发布版本。
- 共享框架承载插件：给三件套（EM.Core / EM.Data / EM.Sdk）与 BouncyCastle 一个带 BepInEx 插件身份的宿主，游戏内全局只保留一份。
- 各功能插件通过 `[BepInDependency(EM.Framework.FrameworkInfo.Guid)]` 显式声明依赖，从结构上消除同名程序集互相抢占导致的 MissingMethodException / TypeLoadException。
- 加载时打一行版本日志，本身不安装任何钩子。
- 提供各功能插件共用的兼容自检（Unity 版本 + global-metadata.dat 哈希双门禁）。
