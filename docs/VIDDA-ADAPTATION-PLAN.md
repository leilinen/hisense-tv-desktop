# Vidda 适配开发计划

## 1. 文档目的

本文定义 `hisense-tv-desktop` 适配海信 / Vidda 电视的开发范围、架构边界、实施顺序和验收标准。

适配目标不是把现有小米私有参数替换成海信参数，而是保留归藏电视现有桌面体验，为不同电视平台建立独立、可验证、可安全降级的适配层。

目标电视的系统类型尚未确认，因此计划同时覆盖两条路线：

1. **海信 / Vidda Android TV**：继续运行现有 Android APK，增加海信 Android 平台适配器。
2. **原生 VIDAA OS**：现有 APK 无法直接运行，需要新增 VIDAA Web 电视端和独立受限 companion。

每个兼容结论必须绑定到具体的型号、地区、系统类型和固件版本。不能仅凭“海信”或“Vidda”品牌名称推断能力。

---

## 2. 产品目标与非目标

### 2.1 第一版目标

保留归藏电视现有核心体验：

- 照片壁纸、纯色背景和壁纸轮播；
- 毛玻璃焦点、圆角和现有视觉层级；
- 纵向列表和横向 Dock；
- 遥控器方向键、确认键和返回键交互；
- 首页设置预览；
- 时间、天气和默认关闭的 Codex 实验；
- 1–9 个首页入口、排序和隐藏；
- 现有小米适配能力不回归。

Vidda 第一版新增能力：

1. 发现并显示目标电视上的应用；
2. 以稳定身份启动应用；
3. 发现 HDMI、TV 等真实信号源；
4. 以稳定身份切换信号源；
5. 显式返回 Vidda 原生桌面；
6. companion 离线、协议不兼容或厂商组件不可调用时安全降级；
7. 保留原厂桌面、Home、直播、投屏、系统设置和 HDMI，不通过禁用系统组件实现接管。

### 2.2 第一版非目标

第一版不实现：

- 持续监听并抢占 Home 键；
- 参考项目中每三秒重新拉起自定义桌面的 Home watcher；
- 任意遥控器按键注入；
- 音量、电源、账号或工厂菜单控制；
- 任意 MQTT topic、payload 或原始 RPC；
- 默认开启的开机自动进入；
- 通过无障碍、悬浮窗、常驻服务或修改默认 Home 模拟系统桌面；
- 将 RemoteNow 私钥、配对码或固定控制凭据放入 APK、网页资源或 Git；
- 将一台 Vidda 电视的字段、source ID 或状态值直接外推到其他型号。

---

## 3. 平台识别与适配分支

开发前必须先确认目标电视运行 Android TV 还是原生 VIDAA OS。

### 3.1 Android TV 判据

满足以下大部分条件时，进入 Android 路线：

- 电视提供 Android ADB 调试；
- `getprop ro.build.version.sdk` 返回有效 API 级别；
- 可以使用 `pm list packages`、`dumpsys package` 和 `cmd package`；
- 可以安装普通 Android APK；
- 应用、桌面和系统设置以 Android package/component 表达。

只读采集示例：

```sh
adb devices -l
adb -s TV_SERIAL shell getprop ro.product.manufacturer
adb -s TV_SERIAL shell getprop ro.product.brand
adb -s TV_SERIAL shell getprop ro.product.model
adb -s TV_SERIAL shell getprop ro.product.name
adb -s TV_SERIAL shell getprop ro.product.device
adb -s TV_SERIAL shell getprop ro.build.version.release
adb -s TV_SERIAL shell getprop ro.build.version.sdk
adb -s TV_SERIAL shell getprop ro.build.display.id
adb -s TV_SERIAL shell wm size
adb -s TV_SERIAL shell wm density
```

### 3.2 原生 VIDAA OS 判据

满足以下大部分条件时，进入 VIDAA Web 路线：

- 不能安装普通 Android APK，或没有可用的 Android ADB 环境；
- TV Browser 支持 VIDAA channel / Web 应用；
- 固件可能提供 `hisense://debug` 一类调试安装入口；
- 应用和信号源可通过 RemoteNow 风格局域网协议发现；
- 电视端运行环境是旧版 Chromium / OMI 浏览器，而不是 Android Activity。

必须实机记录：

- 完整型号和地区 SKU；
- VIDAA OS 与固件版本；
- 浏览器完整 User-Agent 和可用 Web API；
- 逻辑渲染尺寸和刷新率；
- 遥控器各按键的 `event.key`、`keyCode` 和系统行为；
- SSDP 响应和设备描述信息；
- 应用列表、信号源列表和电视状态的脱敏响应样本；
- 回到原生桌面的实际返回键、退出 channel 或协议入口。

### 3.3 采集约束

第一轮采集只读，不执行：

- `settings put` / `settings delete`；
- 修改默认 Home 或输入法；
- 禁用、卸载或清除原厂应用；
- 调用未经检查的厂商组件；
- 注入遥控器按键；
- Root、刷机或 remount；
- 为采集而强制重启电视。

原始 IP、MAC、序列号、ADB 密钥、账号、配对码和个人日志不得写入仓库。

---

## 4. 共享架构改造

无论最终进入哪条路线，都应先把厂商能力与首页 UI 分离。

### 4.1 平台适配器

将当前 `XiaomiControls` 承担的职责拆成品牌无关接口。概念接口如下：

```text
TvPlatformAdapter
  identity()
  capabilities()
  defaultEntries()
  openInput(sourceId)
  showInputPicker()
  openSystemSettings(kind)
  openStockHome()
  labelForApplication(applicationId)
  bootIntegration()
```

首批实现：

- `GenericAndroidTvAdapter`
- `XiaomiTvAdapter`
- `HisenseAndroidTvAdapter`

原生 VIDAA 不直接实现 Android 接口，而是通过受限 companion provider 向电视 Web 前端提供同等能力模型。

### 4.2 能力模型

UI 不按品牌名决定是否显示功能，而是读取能力：

```text
DeviceCapabilities
  canListApps
  canLaunchApps
  canListSources
  canSwitchSources
  canOpenStockHome
  canOpenSystemSettings
  canShowInputPicker
  canConfigureBootEntry
```

能力未知、组件不可用或 companion 离线时，隐藏入口或显示“此固件尚未验证”，不能尝试另一个品牌的私有参数。

### 4.3 首页项目模型

将固定包名和 `hdmi:1` 一类字符串逐步升级为稳定模型：

```text
HomeEntry
  stableId
  type: APP | INPUT | STOCK_HOME | SYSTEM_ACTION
  providerId
  title
  subtitle
  iconKey
  launchTarget
```

身份规则：

- Android 应用：package/component；
- 小米 HDMI：Xiaomi provider 内部稳定 ID；
- VIDAA 应用：设备身份 + remote app stable ID；
- VIDAA 信号源：设备身份 + remote source stable ID；
- 展示名称只用于 UI，不作为启动身份。

### 4.4 现有偏好迁移

迁移必须无损：

- 旧 package 项转换为 Android app stable ID；
- 旧 `hdmi:1`、`hdmi:2` 继续归 Xiaomi provider 所有；
- 未识别的旧项目先保留，不自动删除；
- VIDAA 项保存 provider 和设备标识，避免换电视后误启动；
- 不改变现有壁纸、外观、天气、Codex 和开机开关偏好键，除非另有迁移测试。

### 4.5 UI 保持范围

Android 路线继续复用：

- `HomeSurface`
- `WallpaperMaterial`
- `DockShelf`
- `DockScroll`
- `HomeSettings`
- 当前毛玻璃缓存和显式焦点图

不得为了 Vidda 适配把现有 Android 主界面替换为 WebView。

原生 VIDAA 路线按照相同设计规格重写电视端，但不与 Android View 代码强行共享运行实现。

---

## 5. 小米现有能力的回归保护

Vidda 改造不能改变当前已验证的小米路径。

`XiaomiTvAdapter` 必须保留：

- `MiTV-MTSU0` 设备保护；
- HDMI 1/2 对应当前固件 `input=23/24`；
- 小米输入源广播；
- 小米系统设置和 Wi-Fi 设置入口；
- `com.mitv.tvhome.MainActivity` 公开 alias；
- 目标组件的 enabled、exported 和 permission 检查；
- 当前开机自动进入的机型、开关、boot count、时间窗和每次启动一次策略。

明确禁止：

- 将海信型号加入 `XiaomiControls.enabled()`；
- 将 VIDAA source ID 填入小米 HDMI extra；
- 小米调用失败后自动尝试控制局域网中的 VIDAA 电视；
- 使用 Vidda 适配改变小米 Home、返回键、CEC 或开机行为。

---

## 6. Android Vidda 实施路线

### 6.1 系统组件采集

对目标固件采集并核对：

- 当前 HOME resolver 和全部 HOME 候选；
- 原厂桌面的 Activity 或 activity-alias；
- 系统设置和 Wi-Fi 设置入口；
- 投屏、相册、直播和媒体中心应用；
- TV Input 服务；
- HDMI / 外部信号源相关 Activity、Receiver 或 Service；
- 输入源面板；
- 各组件的 enabled、exported 和 permission；
- 是否要求 signature、privileged、system UID 或厂商白名单。

候选组件只有在普通 APK 可以合法调用时才能进入适配实现。受平台签名保护的能力应明确报告“不支持普通安装包”，不得尝试绕过。

### 6.2 应用发现和启动

继续使用 Android 标准能力：

- `CATEGORY_LEANBACK_LAUNCHER`
- `CATEGORY_LAUNCHER`
- package/component 作为稳定身份
- 优先 `getLeanbackLaunchIntentForPackage()`
- 回退 `getLaunchIntentForPackage()`

不得退化为按显示名称或 substring 查找应用。

需要额外验证：

- 同一 package 有多个 launcher Activity；
- targetSdk 34 下的 package visibility；
- 系统包和用户包的可见性；
- 应用安装、升级、禁用和卸载后的目录刷新；
- 外部应用返回后的焦点恢复。

### 6.3 HDMI 和 TV Input

优先调查标准 `TvInputManager` 和厂商公开入口。信号源模型至少包含：

```text
SourceDescriptor
  providerId
  sourceId
  displayName
  kind
  signalState: YES | NO | UNKNOWN
  currentState: CURRENT | NOT_CURRENT | UNKNOWN
```

约束：

- HDMI 数量和名称来自目标电视；
- 不硬编码 HDMI1/2/3；
- 不复制小米 `input=23/24`；
- source ID 必须与设备和 provider 绑定；
- 组件存在不等于切换语义已经验证；
- 切换后应重新读取状态；
- 没有确认机制时显示“请求已发送，结果未确认”。

### 6.4 返回 Vidda 原生桌面

`HisenseAndroidTvAdapter.openStockHome()` 应：

1. 查询 `ACTION_MAIN + CATEGORY_HOME`；
2. 排除本应用；
3. 优先使用目标固件上实测过的公开 Activity/alias；
4. 调用前检查 enabled、exported 和 permission；
5. 多个候选时不静默猜测；
6. 不修改系统默认 HOME；
7. 启动失败时只提示，不循环重试。

首页提供明确的“Vidda 原生桌面”入口。主层返回键是否直接回原厂桌面由目标 profile 和实机行为决定，但无论如何都不能阻止遥控器 Home 键。

### 6.5 Manifest 和构建

- 将小米 `<queries>` 留在小米实现所需范围；
- 只添加实机确认有必要的海信包可见性声明；
- 不使用 `QUERY_ALL_PACKAGES` 代替精确查询；
- 不新增无障碍、悬浮窗、系统设置写权限或常驻后台服务；
- 如果不同 Android 厂商构建需要明显不同的 Manifest、资源或依赖，再评估 product flavor；
- 不为第一步抽象过早增加 flavor 维护成本。

---

## 7. 原生 VIDAA OS 实施路线

### 7.1 目录建议

原生 VIDAA 与 Android APK 是不同部署目标，建议使用独立目录：

```text
vidaa/
  launcher/
    index.html
    app.js
    styles.css
    assets/
  companion/
    discovery.py
    protocol.py
    catalog.py
    server.py
    tests/
```

VIDAA companion 不与现有 Codex companion 的 RPC 或权限边界混合。可以共享通用的 TLS、token、请求大小和错误处理工具，但控制接口应独立。

### 7.2 电视端视觉实现

参考目标固件浏览器能力选择 Canvas 或保守 DOM/CSS。旧 VIDAA 浏览器可能对复杂 CSS 和 DOM 支持有限，因此第一实现应优先保证稳定渲染。

需要保持：

- 1920×1080 设计坐标系并按实际画布缩放；
- 照片、渐变和纯色背景；
- 低分辨率预模糊背景层；
- 毛玻璃焦点框、圆角和现有透明度层级；
- 列表和 Dock；
- 首尾不循环；
- 焦点项完整可见；
- 长按确认不重复启动；
- 时间、天气和设置面板；
- companion 离线状态；
- 当前应用/信号源状态。

可提供性能档位：

- 标准毛玻璃；
- 低分辨率模糊和较少动画；
- 无模糊的半透明纯色底板。

性能档位应由实机帧时间和内存决定，不能只按品牌名自动选择。

### 7.3 受限 companion

参考 `Zfinix/vidaa-launcher` 可以确认的协议研究线索：

- SSDP 发现 Hisense / VIDAA；
- RemoteNow 风格 TLS MQTT；
- 应用列表；
- 信号源列表；
- 电视状态；
- 应用启动；
- 信号源切换。

参考仓库固定版本：

- <https://github.com/Zfinix/vidaa-launcher/tree/ca622fdef41e66564e46ddd816ef1714453270a2>

但其实现不能直接成为本项目安全基线。新的 companion 必须：

- 结构化解析 SSDP 响应；
- 收集全部候选设备，并由用户选择；
- 以稳定设备身份而不是首个 IP 作为目标；
- 限制 description URL 为本地私网目标；
- 配对响应匹配 topic、action、设备和 session；
- 请求使用单独 correlation 或等价的陈旧响应防护；
- 不将无关 broadcast 当作当前请求响应；
- 使用 monotonic timeout 和有限退避；
- 不向网页暴露 RemoteNow 证书、私钥、账号或 MQTT 细节；
- 不提供任意遥控键、音量、电源、shell 或原始 RPC；
- 不把 VIDAA 私钥打包进 APK 或网页；
- 不按应用名称模糊匹配。

如果设备协议本身无法验证电视服务端证书，应将这一限制隔离在本地协议 adapter 中，记录风险，并通过设备选择、私网限制和上层认证降低影响。不能因此关闭电视网页到 companion 的身份认证。

### 7.4 Companion API

第一版只暴露有限接口：

```text
GET  /api/capabilities
GET  /api/apps
GET  /api/sources
GET  /api/state
POST /api/apps/{opaqueId}/launch
POST /api/sources/{opaqueId}/activate
POST /api/system/stock-home
```

要求：

- HTTPS 或目标环境中等价的加密通道；
- 独立随机 token；
- 证书 pin 或等价的首次配对身份绑定；
- 禁止重定向；
- 限制请求和响应大小；
- 状态修改只使用 POST；
- 拒绝浏览器跨站 Origin；
- 客户端只传 opaque ID；
- 服务端从当前设备目录解析完整 descriptor；
- 陈旧 descriptor、错误设备和错误 session 必须拒绝。

不得提供：

- `/rpc`
- `/mqtt`
- 任意 topic/action/payload
- 任意按键
- 音量和电源
- Home watcher
- 任意 URL 启动

### 7.5 应用和信号源目录

```text
AppDescriptor
  opaqueId
  deviceId
  remoteAppId
  name
  launchable

SourceDescriptor
  opaqueId
  deviceId
  remoteSourceId
  displayName
  kind
  signalState: YES | NO | UNKNOWN
  currentState: CURRENT | NOT_CURRENT | UNKNOWN
```

解析器必须处理固件差异：

- `is_signal = "1"`
- `is_signal = 1`
- `is_signal = true`
- 字段不存在
- source 名称因 CEC 设备或用户重命名而变化
- app/source 列表顺序变化
- 当前状态使用未知 `statetype`

未知数据保留为 `UNKNOWN`，不能自动解释为无信号或未激活。

### 7.6 返回 Vidda 原生桌面

回退是基础能力，必须先于应用启动和切源完成。

#### 返回键层级

- 设置页或二级面板打开：返回上一级；
- 首页主层：不再消费系统返回，允许 VIDAA 退出 channel 回到原生桌面；
- 实机验证 `GoBack`、`Back`、`BrowserBack`、461、10009 等值；
- 不像参考项目那样在主层把返回键改成“聚焦第一项”。

#### 显式入口

首页控制区提供“Vidda 原生桌面”：

- 优先使用经实机验证的 channel close / exit 能力；
- 如果协议提供稳定的原厂 launcher descriptor，可以按 stable ID 启动；
- 不按名称 substring 搜索；
- 不仅凭 `statetype == "remote_launcher"` 推断可启动入口。

#### 离线回退

companion 离线时：

- 页面仍显示壁纸、时间和设置；
- 显示非阻塞的离线状态；
- 返回键和 Home 键仍可离开；
- 不连续弹错；
- 不循环拉起自身；
- 不妨碍用户停留在原生桌面、设置或 HDMI。

### 7.7 不采用 Home watcher

参考项目的 `home_watch.py` 每三秒检查电视状态，并在原生 launcher 出现后重新打开自定义应用。该行为：

- 不是系统默认 Launcher；
- 会短暂显示原生桌面；
- 依赖外部主机持续在线；
- 可能与设置、直播、HDMI 和用户主动回到原生桌面的行为冲突；
- 与本项目“可靠回退 Vidda 原生桌面”的目标冲突。

因此第一版禁止采用。

未来若有明确需求，应作为独立、默认关闭的实验功能，并另行设计：

- 明确开关和醒目状态；
- 物理停止方式；
- 最大重试次数与指数退避；
- 原生桌面停留豁免；
- 设置、直播、投屏和 HDMI 豁免；
- companion 离线自动停用；
- 完整恢复和卸载步骤。

---

## 8. 安全与隐私边界

### 8.1 私密材料

以下内容只存储在仓库外的本地私有目录：

- RemoteNow 证书和私钥；
- 配对码和设备令牌；
- ADB 私钥；
- 电视 IP、MAC、序列号和账号信息；
- 原始抓包和可能包含个人输入的日志；
- 从电视提取的厂商 APK。

仓库中只保留：

- 脱敏 fixture；
- 包名、组件名、字段 schema；
- 型号、地区和固件版本；
- 不可反推个人设备身份的证书摘要；
- 明确授权公开的测试结果。

### 8.2 权限边界

第一版不新增：

- `QUERY_ALL_PACKAGES`
- 无障碍服务
- 悬浮窗
- `WRITE_SETTINGS`
- 录音权限
- 常驻前台服务
- 任意 shell/MCP/文件访问

厂商组件要求 signature/privileged 权限时，不通过伪造权限名、自签名或绕过校验解决。需要厂商签名的能力必须走正式预装或厂商集成流程。

### 8.3 网络边界

不接受参考实现中的以下模式：

- `0.0.0.0` 上无鉴权 HTTP 控制服务；
- 使用 GET 执行启动或切源；
- 关闭网页到 companion 的证书和身份校验；
- 暴露原始 MQTT；
- 固定 MQTT 凭据和伪造设备身份成为上层 API；
- 任意客户端向电视发送遥控器键；
- 以 app/source 显示名称作为执行目标。

---

## 9. 分阶段开发切片

### 切片 0：平台识别和实机 fixture

#### 前置条件

- 有一台明确授权测试的 Vidda 电视；
- 知道型号、地区和固件版本；
- 设备拥有者可确认 ADB 或 VIDAA 调试授权。

#### 工作内容

- 判定 Android TV 或原生 VIDAA OS；
- 采集显示、遥控器和系统身份；
- Android：采集 package/component 和 TV Input；
- VIDAA：采集脱敏 SSDP、app list、source list 和 state；
- 验证返回原生桌面的真实路径。

#### 禁止事项

- 不修改系统设置；
- 不切换默认 Home；
- 不接管 Home；
- 不提交原始私密数据。

#### 完成判据

- 平台类型明确；
- 实机 fixture 可离线解析；
- 原生桌面回退路径有证据；
- 未知字段不会导致解析器崩溃。

### 切片 1：抽象现有 Android 能力

#### 工作内容

- 引入平台 adapter 和 capability；
- 用 Xiaomi adapter 包装现有实现；
- 引入稳定 `HomeEntry` 模型；
- 保持现有 UI 和偏好行为。

#### 完成判据

- 小米 HDMI、设置、输入源、原桌面和开机行为无变化；
- 非匹配设备不调用小米私有接口；
- Manifest 权限不扩张；
- 现有测试和构建通过。

### 切片 2：原厂桌面回退

#### 工作内容

- Android：解析并启动已验证的 Vidda HOME alias；
- VIDAA：主层返回退出 channel，并提供显式原生桌面入口；
- 定义离线和失败行为。

#### 完成判据

- companion 在线和离线时均能退出；
- 不修改默认 Home；
- 不发生自定义桌面与原厂桌面循环；
- Home 键保持系统行为。

### 切片 3：只读应用与信号源目录

#### 工作内容

- 显示 apps、sources 和当前状态；
- 加入 UNKNOWN 状态；
- 对目录更新执行稳定 ID 对账；
- 网络刷新不重置焦点。

#### 完成判据

- 不执行应用启动和切源；
- companion 超时不阻塞 UI；
- 目录变化不导致 index 越界或错误复用 ID；
- 至少两组固件 fixture 测试通过，或明确只有单设备证据。

### 切片 4：按稳定 ID 启动应用

#### 工作内容

- 只允许目录中当前有效的 app descriptor；
- 状态修改使用认证 POST；
- 启动后重新读取状态；
- 失败不自动重试。

#### 完成判据

- 不按名称 substring 匹配；
- 重名应用不会误启动；
- 陈旧、错误设备或错误 session 的 ID 被拒绝；
- 返回桌面后恢复原焦点。

### 切片 5：按 descriptor 切换信号源

#### 工作内容

- 只允许当前目录中的 source descriptor；
- 切换后刷新 state；
- 区分成功、失败和未确认；
- 测试无信号、CEC 重命名和源数量变化。

#### 完成判据

- 不接受原始 source payload；
- 不硬编码 HDMI 数量；
- 不复用小米 source ID；
- 切源失败不会循环重试或抢回桌面。

### 切片 6：视觉和性能验收

#### 工作内容

- 720p、1080p、4K；
- 1、6、9 个入口；
- 列表和 Dock；
- 三档毛玻璃和纯色；
- companion 在线、离线和恢复；
- 长按、快速按键和焦点恢复；
- 首帧、帧时间、内存和长期运行。

#### 完成判据

- 不出现双焦点或焦点丢失；
- 网络操作不在 UI 主线程；
- 低性能模式仍保持可读焦点；
- 长期运行没有持续内存增长；
- 视觉变化有目标电视实机证据。

### 切片 7：评估开机自动进入

开机自动进入不属于第一版。只有在目标固件提供合法、可恢复且不破坏原生桌面的机制后才评估。

要求：

- 默认关闭；
- 用户显式开启；
- 不擅自写入系统目标；
- 完整开机和待机唤醒分开处理；
- 每次完整启动最多尝试一次；
- 过晚调用不抢占当前画面；
- 记录并恢复原设置；
- 不复制小米 `start_3rd_app`；
- 不使用 Home watcher 代替开机机制。

---

## 10. 测试矩阵

### 10.1 构建和静态边界

| 项目 | Android APK | VIDAA Web | VIDAA companion |
| --- | --- | --- | --- |
| 构建/静态检查 | Debug、Release、lint | HTML/JS 静态检查 | Python 测试和静态检查 |
| 单元测试 | adapter、模型、迁移 | 导航和解析 | discovery、协议、API |
| 高权限检查 | 不新增高权限 | 不适用 | 不提供 shell/RPC |
| secrets 扫描 | 必须 | 必须 | 必须 |
| fixture 脱敏 | 按需 | 按需 | 必须 |

### 10.2 设备矩阵

至少覆盖：

- 已验证的小米 `MiTV-MTSU0` 固件；
- 同型号不同小米固件，验证保护边界；
- 非匹配 Android TV；
- 目标 Vidda 设备；
- 条件允许时第二种 Vidda 型号、固件或地区版本；
- 同一局域网内两台 VIDAA；
- 无 VIDAA 设备的普通网络。

### 10.3 应用与信号源

- 应用重名；
- 同 package 多 launcher Activity；
- 应用安装、升级、禁用和卸载；
- source 数量和顺序变化；
- HDMI 无信号和有信号；
- CEC 设备连接、断开和重命名；
- TV tuner、AV、USB 等非 HDMI source；
- 切换成功、失败、超时和无 ACK；
- app/source 陈旧 descriptor；
- 连接另一台电视后的设备身份隔离。

### 10.4 UI 和遥控器

组合覆盖：

- 列表 / Dock；
- 文字 / 图标加文字 / 图标；
- 文字焦点 / 毛玻璃；
- 壁纸 / 纯色；
- 1 / 6 / 9 个入口；
- companion online / offline / reconnecting。

动作覆盖：

- 单步上下左右；
- 长按方向键；
- 快速交替按键；
- Center / Enter；
- Back；
- Home；
- 打开应用并返回；
- 切换信号源；
- 打开设置并逐层返回；
- 原生桌面显式入口；
- companion 超时期间继续移动焦点。

验收：

- 无双焦点；
- 无焦点丢失；
- 长按确认不重复启动；
- 网络刷新不偷焦点；
- 毛玻璃位置与焦点一致；
- 原生桌面回退不形成循环。

### 10.5 网络和安全

- 错 token；
- 错证书 pin；
- companion IP 改变；
- 缓存 IP 指向另一台设备；
- 浏览器跨站 Origin；
- 超大 body 和 response；
- redirect；
- 任意 command/RPC 探测；
- 任意 MQTT payload 探测；
- stale response；
- unrelated broadcast；
- 多客户端并发；
- 电视重启、待机和网络切换；
- companion 离线和重启。

预期：拒绝不可信请求，UI 显示离线或未确认，本地 Android 应用列表和原厂桌面回退不受影响。

### 10.6 性能

- 首次和二次启动；
- 首帧时间；
- 焦点移动平均和 95 分位帧时间；
- 模糊生成耗时；
- 壁纸切换帧率；
- Java/native/graphics 或浏览器内存；
- 低内存/页面重载恢复；
- 1 小时和 8 小时运行；
- 网络断开重连的 CPU 和日志频率；
- 720p、1080p、4K 的内存和清晰度。

---

## 11. 第一版验收标准

只有同时满足以下条件，才能声明某个 Vidda 型号和固件“已适配”：

1. 系统类型、型号、地区和固件版本已记录；
2. 归藏电视视觉和遥控器焦点行为达到目标设备验收标准；
3. 应用目录来自目标电视的真实发现结果；
4. 应用按稳定 ID 启动，不按名称猜测；
5. 信号源来自目标电视真实目录；
6. 信号源切换区分成功、失败和未确认；
7. 首页有明确的 Vidda 原生桌面入口；
8. companion 离线时仍能返回原生桌面；
9. 返回键和 Home 键不会造成桌面循环；
10. 原厂桌面、直播、投屏、系统设置和 HDMI 未被禁用；
11. 仓库和发布产物不包含私钥、配对码或个人日志；
12. 不暴露无鉴权控制 API、原始 MQTT/RPC 或任意按键；
13. 小米已验证能力和通用 Android 能力没有回归；
14. 未验证项、权限限制和恢复方式已写入兼容性文档。

---

## 12. 兼容性记录模板

每个型号和固件单独填写：

```text
设备身份
- 品牌：
- 型号：
- 地区/SKU：
- 系统类型：Android TV / VIDAA OS
- Android API 或 VIDAA 版本：
- 固件/build ID：
- 安全补丁或浏览器 UA：

显示与输入
- 逻辑尺寸：
- density/缩放：
- 当前刷新率：
- 遥控器型号：
- 方向/确认/返回/Home 行为：
- 输入源键行为：
- 语音键行为：

应用
- 发现机制：
- 已验证启动的应用：
- 重名/多入口行为：
- 失败回退：

信号源
- 发现机制：
- source ID 稳定性：
- HDMI/TV/AV 列表：
- 无信号状态：
- CEC 重命名：
- 切换确认机制：

原生桌面
- 原厂桌面身份：
- 显式入口：
- 返回键路径：
- Home 键路径：
- companion 离线回退：

安全与权限
- Android 组件 exported/enabled/permission：
- VIDAA 配对和证书限制：
- 普通安装包无法获得的能力：

测试产物
- Android APK commit/version/signature source：
- VIDAA launcher commit：
- companion commit/runtime/dependency version：
- 测试日期：

结论
- 已验证：
- 未验证：
- 已知限制：
- 恢复步骤：
```

不要在模板中填写电视 IP、MAC、序列号、ADB 公钥、配对码、账号或私钥。

---

## 13. 参考资料与使用边界

### 本仓库

- [README](../README.md)
- [AI 适配与引导指南](AI-SETUP.md)
- [兼容性与验证边界](COMPATIBILITY.md)
- [开机自动进入](BOOT-START.md)
- [Codex 实验](CODEX.md)
- [设置指南](SETTINGS.md)

### 外部参考

- [`Zfinix/vidaa-launcher` 固定 commit](https://github.com/Zfinix/vidaa-launcher/tree/ca622fdef41e66564e46ddd816ef1714453270a2)

该参考项目是：

- VIDAA Web/Canvas 电视端；
- Python 局域网服务器；
- RemoteNow 风格 TLS MQTT 控制；
- 轮询电视状态后重新打开自定义 channel 的 Home watcher。

它不是 Android Launcher，也没有 Android Manifest、Activity、HOME intent-filter 或 Android 系统签名集成。

本项目可以参考：

- SSDP 和 RemoteNow 协议研究线索；
- app/source/state 分层；
- 低分辨率渲染和预模糊思路；
- VIDAA 浏览器遥控器按键候选值。

本项目不采用：

- `home_watch.py` 持续抢回前台；
- 关闭 TLS 主机名和服务端证书验证作为通用安全策略；
- 无鉴权、监听所有接口的 HTTP 控制服务；
- 使用 GET 执行状态修改；
- 固定 MQTT 凭据和伪造设备身份进入上层 API；
- 任意按键透传；
- 按应用名称 substring 启动；
- 把单台电视的字段和 source ID 宣称为整个 VIDAA 系列兼容。

---

## 14. 实施原则摘要

1. 先识别系统类型，再选择 Android 或 VIDAA 路线。
2. 保留归藏电视视觉，不保留小米私有实现的耦合结构。
3. 小米、海信 Android 和原生 VIDAA 使用独立 adapter/provider。
4. 先完成原生桌面回退，再实现应用启动和信号源切换。
5. 能力必须来自实机发现和权限验证，未知即不支持。
6. 应用和信号源使用稳定 ID，显示名称不作为执行身份。
7. companion 只暴露有限、认证、可审计的操作。
8. 不通过 Home watcher、无障碍、悬浮窗或禁用系统桌面实现接管。
9. companion 离线不能妨碍用户回到 Vidda 原生桌面。
10. 每项兼容结论都绑定具体型号、地区、固件和测试产物。
