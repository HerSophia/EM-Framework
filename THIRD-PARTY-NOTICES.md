# 第三方声明（THIRD-PARTY-NOTICES）

> English note follows the Chinese section.

EM.Framework 本身按 MIT 协议发布（见 LICENSE）。它在运行时依赖或随包分发以下第三方组件，各自的版权与许可归其原作者所有。

## 随包分发的组件

| 组件 | 版本 | 许可 | 说明 |
| ---- | ---- | ---- | ---- |
| BouncyCastle.Cryptography | 2024.5 起构建 | MIT | 随 EM.Framework 目录分发的 `BouncyCastle.Cryptography.dll`，供 EM.Data 的 AES-GCM 使用。 |

BouncyCastle.Cryptography 版权归 The Legion of the Bouncy Castle Inc. 所有，按 MIT 许可发布。许可原文见 https://www.bouncycastle.org/licence.html 。

## 运行时依赖（不随本包分发，由 BepInEx 环境提供）

| 组件 | 许可 | 说明 |
| ---- | ---- | ---- |
| BepInEx | LGPL-2.1 | 插件加载器，由用户单独安装（BepInExPack_EM2026）。本框架仅依赖它，不修改其代码。 |
| HarmonyLib（0Harmony） | MIT | 运行时打补丁库，随 BepInEx 提供。 |

BepInEx 版权归 BepInEx 团队所有，按 LGPL-2.1发布，源码见 https://github.com/BepInEx/BepInEx 。
HarmonyLib 版权归 Andreas Pardeike 所有，按 MIT 发布，源码见 https://github.com/pardeike/Harmony 。

---

# Third-Party Notices (English)

EM.Framework itselfis released under the MIT License (see LICENSE). It depends on or redistributes the following third-party components at runtime. Copyrights and licenses belong to their respective authors.

## Redistributed components

| Component | Version | License | Notes |
| ---- | ---- | ---- | ---- |
| BouncyCastle.Cryptography | 2024.5 build | MIT | `BouncyCastle.Cryptography.dll` shipped in the EM.Framework folder, used by EM.Data for AES-GCM. |

BouncyCastle.Cryptography isCopyright (c) The Legion of the Bouncy Castle Inc., released under the MIT license. See https://www.bouncycastle.org/licence.html .

## Runtime dependencies (not shipped in this package; provided by the BepInEx environment)

| Component | License | Notes |
| ---- | ---- | ---- |
| BepInEx | LGPL-2.1 | Plugin loader installed separately by the user (BepInExPack_EM2026). This framework depends on it and does not modify its code. |
| HarmonyLib (0Harmony) | MIT | Runtime patching library shipped with BepInEx. |

BepInEx is licensed under LGPL-2.1, source at https://github.com/BepInEx/BepInEx .
HarmonyLib is Copyright (c) Andreas Pardeike, licensed under MIT, source at https://github.com/pardeike/Harmony .
