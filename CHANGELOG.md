# CHANGELOG — EM.Framework

版本号遵循语义化版本。适配游戏：Esports Manager 2026 Steam 正版补丁 1.0.2 小热修复（2026-07-13，Unity 6000.3.12f1 / IL2CPP metadata v39）。

## 0.1.7（2026-07-15）

- EM.Core 补充：声望成长内核新增按本届综合 Rating 分层的加成逻辑，供 BetterReputation 使用。在原有「赛事级别 + 名次 + MVP」锚定之上，按选手本届综合 Rating 追加档位加成（达到阈值再顶一到两档），并与 MVP 顶档合并后受一个上限约束。加成默认开启、可关，关掉即退回原有锚定。属新增能力，对既有插件无影响、无破坏。
- 框架结构与加载行为未变，各插件依赖方式不变，直接替换本框架五个文件即可。

## 0.1.6（2026-07-14）

- EM.Sdk 新增：单场结算钩子 MatchHooks，把「一场比赛刚结算完成」暴露为可订阅事件 OnMatchCompleted，插件不必自己写 Harmony。同时挂两条结算路径——MatchesEngine.EndMatch（玩家侧比赛）与 MatchesEngine.AutoGenerateMatch（覆盖全部 AI 对 AI 比赛，以及跳过观看直接模拟的比赛）；上下文带来源标记，供订阅方按 match id 去重。属新增能力，对既有插件无影响、无破坏。供 BetterMvp 单场 MVP 接管使用。
- EM.Core 补充：登记 MatchesEngine.AutoGenerateMatch 方法常量，供上述钩子按名反射定位，不硬编码地址。
- 框架结构与加载行为未变，各插件依赖方式不变，直接替换本框架五个文件即可。

## 0.1.5（2026-07-14）

- EM.Sdk 修复：通用邮件门面 GameMail 创建邮件时，原先给 `DataEmail.Content` 传的是 null，这在游戏里是不允许的——收件箱显示这封邮件时会访问 Content，退出重进读档时也会因 Content 为空而崩溃。本版改为创建一个游戏原生的「知道了」内容体（AcknowledgementContent）作为 Content；万一构造不出，宁可放弃这封邮件也不再发出内容为空的邮件。任何用 GameMail 发信的插件都受益。
- EM.Core 补充：邮件系统相关常量登记（AcknowledgementContent、EmailView 等类型名），供上述修复与存量修复按名反射定位，不硬编码地址。
- 框架结构与加载行为未变，各插件依赖方式不变，直接替换本框架五个文件即可。

## 0.1.4（2026-07-13）

- EM.Sdk 新增：通用大语言模型服务契约。包含客户端接口 `ILlmClient`（提供文本生成与模型列举两项能力）与静态服务定位器 `LlmService`（默认返回空实现，未注册时调用方得到明确的失败结果而非异常）。属新增能力，对既有插件无影响、无破坏。
- EM.Core 新增：大语言模型相关的纯逻辑内核，包括请求与结果模型（`LlmRequest` / `LlmResult` / `LlmModelsResult` 等）、预设配置、失败原因枚举、重试策略，以及两种接口格式（Chat Completions 与 Responses）的请求体构造与响应解析编解码器。这些逻辑不依赖游戏与 Unity，可离线单元测试。供 EM.LlmService 使用。
- 框架结构与加载行为未变，各插件依赖方式不变，直接替换本框架五个文件即可。

## 0.1.3（2026-07-13）

- 适配 Steam 正版 1.0.2 之后的小热修复（2026-07-13，官方说明为转会系统增强与翻译等）：该热修复重新编译了游戏，global-metadata.dat 与 GameAssembly.dll 的指纹随之改变。本版更新兼容自检所用的这两项指纹为热修复的新值，同时保留 1.0.2、1.0.1 与 GoldBerg 指纹于已接受列表，命中任一即通过。
- 框架自身逻辑未变，只是随游戏版本更新校验基准。直接替换本框架五个文件即可。

## 0.1.2（2026-07-13）

- EM.Sdk 修复：通用邮件门面 GameMail 解析 `DataStaffs.GetMyStaffByRole` 时，因游戏内该方法有两个重载而抛 AmbiguousMatchException、导致整个邮件门面停用；改为显式挑选重载，邮件门面恢复正常。任何用到 GameMail 发信的插件都受益。
- EM.Core 新增：声望相关的纯逻辑内核（文案分级器与阈值字段），供 BetterReputation 使用。属新增能力，对既有插件无影响、无破坏。
- 框架结构与加载行为未变，各插件依赖方式不变，直接替换本框架五个文件即可。

## 0.1.1（2026-07-12）

- 适配 Steam 正版补丁 1.0.2：更新兼容自检所用的 global-metadata.dat 与 GameAssembly.dll 指纹为 1.0.2 的值，同时保留 1.0.1 与 GoldBerg 指纹于已接受列表，命中任一即通过。
- 框架自身逻辑未变，只是随游戏版本更新校验基准。

## 0.1.0（2026-07-12）

- 首个对外发布版本。
- 共享框架承载插件：给三件套（EM.Core / EM.Data / EM.Sdk）与 BouncyCastle 一个带 BepInEx 插件身份的宿主，游戏内全局只保留一份。
- 各功能插件通过 `[BepInDependency(EM.Framework.FrameworkInfo.Guid)]` 显式声明依赖，从结构上消除同名程序集互相抢占导致的 MissingMethodException / TypeLoadException。
- 加载时打一行版本日志，本身不安装任何钩子。
- 提供各功能插件共用的兼容自检（Unity 版本 + global-metadata.dat 哈希双门禁）。
