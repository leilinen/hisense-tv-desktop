# Vidda 适配开发计划

- Change-Id: `guizang-tv-20261023-vidda-adaptation-plan`
- 执行者：ZCode
- 授权：用户要求研究本项目适配海信/Vidda 电视的工作范围，参考 `Zfinix/vidaa-launcher` 的实现，并将开发计划写入仓库文档。

新增 [docs/VIDDA-ADAPTATION-PLAN.md](../VIDDA-ADAPTATION-PLAN.md)。计划保留归藏电视现有视觉与遥控器交互，区分两条适配路线：海信/Vidda Android TV 在现有 APK 上抽象平台 adapter；原生 VIDAA OS 新增独立的 VIDAA Web 前端与受限 companion。计划将应用发现/启动、信号源发现/切换和回到 Vidda 原生桌面列为第一版目标，明确不采用参考项目的 Home watcher、无鉴权控制服务、关闭 TLS 校验、固定凭据和按名称模糊启动，也不把小米私有参数复制到海信。

参考项目结论基于其固定 commit `ca622fdef41e66564e46ddd816ef1714453270a2` 的源码审计：该项目是 VIDAA Web 页面加 Python MQTT 桥，不是 Android Launcher。目标 Vidda 电视的系统类型尚未实机确认，文档要求先做只读采集再选择路线；未验证项不写成已兼容。

本次仅新增文档，不修改产品代码、Manifest、构建配置或签名，不安装应用，不执行电视设置操作。验证方式为仓库内相对链接存在性检查、Markdown 空白检查与 Git 差异核对；未运行 Android 构建与实机测试。
