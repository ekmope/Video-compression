# 只读校验报告：对 `final_audit_report.md` 全部可判定条目的独立复核

- 校验时间：2026-09-21
- 校验对象：`C:\Users\20140\WorkBuddy\Worktrees\bug-fixes-996ffc34\workbuddy-bug-fixes-996ffc34-aa62a751\final_audit_report.md`
- 校验基准代码：`C:\Users\20140\WorkBuddy\Worktrees\PiliPlus-main\main-08248366`（下称**工作区**）
- 约束遵守：未修改任何代码；不输出修复方案 / patch / diff / 修复步骤；仅给结论与证据。

---

## 0. 前置声明（影响结论有效性的四点）

**0.1 待校验列表的来源**
用户消息中的「待校验 Bug 列表」只有模板骨架（技术栈、位置、描述、触发条件等字段全部为空）。因此本次以被 `@` 引用的 `final_audit_report.md` 中所有可判定条目为校验对象，共 **24 项**：

- §二 已确认为真：A-01 ~ A-12（12 项）
- §三 已确认为误报：X-01 ~ X-03（3 项）
- §四 已确认为设计如此：D-01 ~ D-04（4 项）
- §五 信息不足：U-01、U-02（2 项）
- §6.2 因 fork 台账产生的 3 项误报：R-10、M-14、L-06

若用户本意是校验另一份列表，需重新提供，本报告结论不适用于该情形。

**0.2 版本基线差异（重要的方法论风险）**
报告中的行号与**快照仓**（审计当时的仓库副本，即 `bug-fixes-996ffc34\workbuddy-bug-fixes-996ffc34-aa62a751`）逐条吻合；但与**工作区**至少有一处已不同源：

| 文件 | 报告/快照仓 | 工作区 |
|---|---|---|
| `lib/plugin/pl_player/view/view.dart` | `:3103` = `if (_stableFrames >= _stableFrameCount && !_surfaceReady) {` | `:3121` 新增局部量 `requiredStableFrames = _hasShownSurface ? _stableFrameCount : 1`，停止分支在 `:3122` |

即工作区比快照仓**多了一处行为等价的重构**（首帧只需 1 个稳定帧，已显示过则需 3 个）。本报告对 A-09 的判定在两侧均成立（见 2.9）。其余抽查的条目（A-05 `rcmd/view.dart:57-63`、A-10 `video_card_h.dart:23/180/403`、D-02 `video/controller.dart:2518-2522`）在两侧完全一致。

**0.3 无法完成的环境验证（不猜测）**
- 工作区无 `.dart_tool/`，未执行过 `flutter pub get`，**不能编译、不能真机运行**；所有「真机实测」类结论一律标注为待补充。
- `developer.android.com` 在当前环境网络不可达，**A-04 引用的官方条文无法独立复核**。
- `dlna_dart` 包源码不在本机 pub 缓存中，**U-01 中 dlna_dart 的 SSDP 实现细节无法在本地核对**。

**0.4 本报告使用的结论口径**
`真实存在` / `误报` / `设计如此` / `环境或配置问题` / `信息不足无法判断`。部分条目原始描述把「代码事实」与「后果」捆在一起断言，本报告对二者分别判定并在「影响」列注明。

---

## 1. 结论总览表

| 编号 | 原始描述（摘要） | 结论 | 置信度 | 关键证据 | 影响 | 待补充信息 |
|---|---|---|---|---|---|---|
| A-01 | iOS 缺 `PrivacyInfo.xcprivacy` | 真实存在（文件确实缺失）；「上架阻塞」属**影响层面待定** | 中 | 全仓 `*xcprivacy` 零命中（含 `ios/Flutter`、`ios/Runner`）；`project.pbxproj` 中也无任何 `PrivacyInfo`/`xcprivacy` 引用 → 从未添加过 | 仅在上架路径成立；本项目实际分发为侧载未签名 IPA（见 3.1） | 需生成 `ios/Pods` 后核对 Pod/SPM 侧清单；需 App Store 当前对宿主清单的强制边界 |
| A-02 | iOS 缺 `NSPhotoLibraryAddUsageDescription` 影响保存相册 | 配置缺失**真实存在**；「保存功能受影响」**不成立/证据不足** | 中（配置事实：高；功能影响：低） | `Info.plist:73-74` 只有 Usage 键；但 `image_utils.dart:86-88` 在 iOS 走 `Permission.photos.request()`（= readWrite 级）；插件 `SaverGalleryPlugin.swift:249` 仅在已授权后 `performChanges` | 合规/审核语义层面；不改写「功能已损坏」的结论 | iOS 真机 + 首次保存（未预先授权路径：`login/view.dart:84` / `pl_player/controller.dart:3288`）实测 |
| A-03 | Bundle ID 误用 `.android`、`DEVELOPMENT_TEAM` 为空 | 代码事实**真实存在**；「影响签名与上架」**不成立** | 高（事实）/ 中（影响） | `project.pbxproj:395/525/549` = `com.PiliMax.android`；`:388/518/542` = `DEVELOPMENT_TEAM = ""` 全部逐行命中 | 实际分发路径下无影响：`.github/workflows/ios.yml:40` 用 `--no-codesign` 构建后 `codesign --sign -` 重签，注释写明 "make AltSign happy" | 无（若未来做 App Store 上架需重新评估） |
| A-04 | Android 缺 `dataExtractionRules` | 真实存在（配置缺失属实） | 中 | `android/app/src/main/res/xml/` **目录不存在**；`AndroidManifest.xml:19-26` 无 `android:dataExtractionRules`；`build.gradle.kts:50` `targetSdk = 37` | 隐私合规：D2D 迁移未受约束（程度随厂商实现） | 官方条文原文未能复核（域名不可达），需人工对照 Android 12 行为变更文档 |
| A-05 | 改字号后列表行高错乱 | 真实存在（机制逐环验证通过） | 中高（机制）/ 中（视觉后果） | `font_setting.dart:143-146`；getx `extension_navigation.dart:516` `appUpdate() => _getxController.update()`；`get_controllers.dart:16-21` → `refresh()`；`get_material_app.dart:191` 根为 `GetBuilder<GetMaterialController>`；`rcmd/view.dart:57-63` 字段级 `late final` + `:23-28` `AutomaticKeepAliveClientMixin` | 已挂载且被 keepAlive 的列表页，行高按旧字号缓存 → 文字放大后裁切/溢出 | 真机验证「改字号 → 返回首页/搜索/UP 主页」是否出现裁切；报告「约 30 处」计数不精确（见 2.5） |
| A-06 | `Grid.smallCardWidth` 为 `static final` | 真实存在（事实）；性质属**已知设计限制** | 高 | `grid.dart:27`；`storage_pref.dart:550-551`；全仓仅 `style_settings.dart:858` 一处写入、无重置入口；`style_settings.dart:860` 应用自己弹「重启生效」 | 设置不即时生效；不造成错乱（报告已限定） | 无（属产品取舍，归类可与 D-01~D-04 统一） |
| A-07 | `CFBundleURLTypes` 结构嵌套错误 | 真实存在 | 高 | `ios/Runner/Info.plist:85-127`：外层 array 首个 dict（`:87-117`）内嵌 `CFBundleURLTypes`（`:95`），内含 12 个 bilibili 域名（`:102-113`）；`:92-93` 注册 `http`/`https` 为 scheme | 影响低：dict 内未知键被忽略，12 条域名条目失效（域名本也不能作 scheme） | 无 |
| A-08 | 关闭弹幕后弹幕层仍逐帧重绘 | 真实存在 | 高 | `danmaku/view.dart:226-231` opacity 置 0 但 `DanmakuScreen` 仍挂载；`:90-98` 仅按播放状态 pause/resume；`pl_player/view/view.dart:341-344` 回调不调 pause；插件 `danmaku_screen.dart:71/282` `_running` 仅 `_pause()` 置 false、`:526/:574` `willChange: _running` | 低-中：弹幕数据管线已停（`:101-104` 提前 return），余每帧重建 + 空图层绘制 | 每帧实际成本未实测（需 profile） |
| A-09 | 播放器表面探针稳定后不再停止 | 真实存在（快照仓与工作区**均成立**） | 高 | 工作区 `:2900` → `_resetStability` `:2999-3007`（`:3005` 无条件重启计时器，`:3006` `preserveSurface` 提前 return 保留 `_surfaceReady=true`）→ 停止分支 `:3122` 的 `!_surfaceReady` 恒假；`:3099` 同源；边界 `:2994` `_stopProbeTimer()` / `:2913` 重置 | 中：同一控制器 + 视口变化（旋转/全屏/小窗缩放）后，100ms 探针持续运行至控制器切换或 dispose | 无（行号需按工作区重定位：报告行号对应快照仓） |
| A-10 | `clickedBvids` 只增不减 | 真实存在（事实）；「缺陷」定性偏弱 | 高（事实）/ 中（定性） | 活路径 `lib/pilimax/forks/.../video_card_h.dart:23` 声明、`:180` 写入、`:403` 读取；全仓仅此 3 处命中，无 remove/clear | 极低：字符串集合 KB 级；且 `:403` 用于会话内「已点过」变色，可能为有意设计 | 需产品意图确认是否为会话标记 |
| A-11 | `DanmakuMergeWorkerClient` 无实例化点 | 真实存在（事实）；性质为**未接线代码** | 高 | 全仓 grep 仅 `worker_client.dart:14,15`；且整个 `lib/pilimax/utils/danmaku_merge/` 目录的 import 全部指向模块内部（`clusterer/normalizer/similarity_matcher/pinyin_encoder/worker_*` 互引），外部零引用 | 无：无运行时行为；弹幕合并实际走 `pages/danmaku/controller.dart:19,105` 的主隔离区实现 | 需确认是否为在建特性（pubspec 已声明 `assets/danmaku_merge/pinyin_dict.txt`，倾向在建） |
| A-12 | `res/raw/keep.xml` 用 `tools:keep="@drawable/*"` | 真实存在（配置事实）；**当前不生效** | 高 | `res/raw/keep.xml:2-3` 内容确实为 `tools:keep="@drawable/*"`；`android/app/build.gradle.kts:94-97` `proguardFiles(...)` 被注释、全文件无 `isShrinkResources` → 资源压缩未启用 | 当前无影响；开启资源压缩后才会生效 | 无 |
| X-01 | 运行期并存两套 Dio/HttpClient 连接池 | **误报**（报告结论正确） | 高 | 全仓 import 图核验：`lib/http/init.dart` 的 import 方恰为 `utils/accounts.dart:1`、`services/download/download_manager.dart:4`、`utils/wbi_sign.dart:9`；`utils/accounts.dart` 的 4 个 import 方（`http/init.dart:10`、`utils/accounts/account.dart:3`、`utils/storage.dart:7`、`pages/later/controller.dart:13`）全部落在 legacy 簇内；`pages/later/widgets/video_card_h_later.dart:12` 是 legacy later controller 的唯一引用方且自身零引用 | 无：legacy 簇无 live 入口，静态 `dio` 不会被初始化 | 无 |
| X-02 | `WRITE_SETTINGS` 属过度声明 | **误报**（报告结论正确） | 高 | `pl_player/view/view.dart:388` 调 `setSystemScreenBrightness`；插件 `ScreenBrightnessAndroidPlugin.kt:335` `setSystemScreenBrightness` → `:337` 判 `canWriteSystemSetting` → `:338-342` 跳 `Settings.ACTION_MANAGE_WRITE_SETTINGS`；`canWriteSystemSetting` `:382-388` 用 `Settings.System.canWrite`（`:384`） | 无：权限为系统亮度手势所必需 | 无 |
| X-03 | `READ_MEDIA_AUDIO/VIDEO` 造成无关弹窗 | **误报**（报告结论正确） | 高 | 全仓仅 `Permission.storage.request()`（`image_utils.dart:87`）与 `Permission.photos.request()`（`:88`）；`Permission.audio.request` / `Permission.videos.request` 零命中；`AndroidManifest.xml:222-223` 仅声明 | 无运行时弹窗；属清单冗余（商店数据安全表单层面） | 无 |
| R-10（§6.2） | `vertical_tabs.dart` build 约 1605 行 | **误报**（报告结论正确） | 高 | 旧路径 2172 非空行（总 2404）；活路径 `pilimax/forks/.../vertical_tabs.dart` 仅 204 非空行（总 218），`Widget build` 只在 `:21`、`:176` | 无 | 无 |
| M-14（§6.2） | 每次 tab 变化生成全部 `GlobalKey` | **误报**（报告结论正确） | 高 | 活路径文件中无 `GlobalKey` 生成逻辑：`_VerticalTabBarState`（`:94-118`）经 `TabController.animation` 监听 + `setState` | 无 | 无 |
| L-06（§6.2） | `shrinkWrap` 属 fork 旧路径 → 误报 | **报告的「误报」判定不成立**（依据错误） | 高 | 原始条目（`mobile_audit_full.md:228`）引用的是**活路径** `pilimax/forks/common/widgets/flutter/vertical_tabs.dart:183`，且该模式在活文件中确实存在（`:186` `shrinkWrap: true`）；同条目的 `dynamics/widgets/vote.dart:90,94,103` 亦为活文件（实际 `:91`、`:104` 均 `shrinkWrap: true`） | 该项应回到「待重查/需重判」，而非计入已确证误报 | 需按活路径逐条重判（另 `:186` 是否构成浪费布局需另行判定） |
| D-01 | `AudioService`/`MediaButtonReceiver` `exported="true"` | **设计如此**（报告结论正确） | 高 | 本仓 `AndroidManifest.xml:174-182`、`:188-195`；上游 example（`Pub\Cache\git\audio_service-e0860cf…/example/.../AndroidManifest.xml:39-47,49-56`）声明同样的 `exported="true"` + `tools:ignore="Instantiatable"` + `MediaBrowserService` intent-filter | 无：上游既定要求 | 报告「逐行一致」措辞偏强（属性顺序/位置不同），结论不受影响 |
| D-02 | `onClose` 在 `isEnteringPip` 提前 return | **设计如此**（报告结论正确） | 中高 | `video/controller.dart:2517-2522`；`pip_overlay_service.dart:124-130` 注释 + `:131-149` `_releaseSavedVideoOwner`（置 `isEnteringPip=false`、按 owner 是否在栈决定 dispose/pause） | 无（有意的所有权交接） | 残余点仍需运行时测量（报告已如实标注） |
| D-03 | 根节点覆盖系统字号 | **设计如此**（报告结论正确） | 高 | `main.dart:464` `TextScaler.linear(Pref.defaultTextScale)`；`:487-508` 两个分支都显式传 `textScaler`；`storage_pref.dart:1357-1358` 默认 1.0；全仓「跟随系统」仅命中 `models/common/theme/theme_type.dart:8`（主题模式，非字号） | 与平台无障碍规范不一致，但为产品取舍 | 无 |
| D-04 | `lib/tcp/live.dart` 被当作死代码 | **设计如此**（报告结论正确） | 高 | `tool/pilimax/fork_map.tsv:4` = `fork lib/tcp/live.dart lib/pilimax/forks/tcp/live.dart` | 无：台账登记的上游快照 | 无 |
| U-01 | 缺 Multicast entitlement 导致 iOS DLNA 不可用 | **升级为：真实存在（机制成立）** | 中 | `Release.entitlements` / `DebugProfile.entitlements` 仅含空 `keychain-access-groups`；Apple 官方文档明确「Your app must have this entitlement to send or receive IP multicast or broadcast on iOS」（iOS 14+，需向 Apple 申请）；`app_pages.dart:200` 无平台门控注册 `/dlna`；`video/controller.dart:2975`、`live_room/controller.dart:711` 直达；`dlna/view.dart` 无平台判断 | 中：iOS 端投屏发现（SSDP）不可用；且该 entitlement 需开发者团队资格，本仓侧载签名路径更无法获得 | iOS 真机 + 局域网 + 可投屏设备实测 SSDP |
| U-02 | 缺 `network_security_config.xml` 导致 cleartext 失败 | **改判为：误报（倾向）** | 中高 | Flutter 官方破坏性变更文档原文：「This only applies to platform native sockets…**Flutter does not enforce any policy at socket level**…If the socket is owned by Dart/Flutter, no policy will be enforced」；flutter#106678 亦证实 `dart:io HttpClient` 不受 `usesCleartextTraffic` 约束。本仓 HTTP 栈为 Dio（`dart:io`），DLNA 走 Dart socket | 低：主 HTTP 栈不受影响；残余不确定仅在 WebView/原生播放器内嵌 http | 若需闭环：真机抓包或 WebView 加载 http 页对照实验 |

---

## 2. 逐项详细说明

### 2.1 A-01　iOS 缺 `PrivacyInfo.xcprivacy`

- **校验方式**：递归检索 + Xcode 工程引用检索。
- **已验证事实**：工作区全仓 `*xcprivacy` 零命中（含 `ios/Flutter`、`ios/Runner`、根目录）；`ios/Runner.xcodeproj/project.pbxproj` 中 `PrivacyInfo|xcprivacy` 零命中。可确认结论是**该文件从未被加入**，而非构建产物中丢失。
- **对照事实**：第三方插件自带清单确实存在，例 `Pub\Cache\hosted\pub.dev\saver_gallery-5.1.0\ios\saver_gallery\Sources\saver_gallery\PrivacyInfo.xcprivacy`，但其 `NSPrivacyAccessedAPITypes` 为空数组 —— 插件清单声明不了宿主 App 自身的 API 使用理由。
- **反证排查**：无法核对 `ios/Pods`（不存在）、无法执行 `pub get`，因此「宿主清单是否必需」取决于构建产物中引擎/依赖清单的覆盖范围，本环境无法闭合。
- **影响判定修正**：报告的「仅上架流程」需要前提。实际分发路径见 3.1 —— iOS 产物是不签名 IPA（CI 侧载用），此前提下该缺失不构成功能或分发阻塞。
- **最小验证方式**：`flutter build ios --release --no-codesign` 后检查 `build/ios/iphoneos/Runner.app/PrivacyInfo.xcprivacy` 是否存在及其内容；同时核对 `Runner.app/Frameworks/*` 内各 manifest。
- **严重程度**：上架路径=阻塞级（若确需宿主清单）；侧载路径=无影响。
- **缺少信息**：Pod/SPM 侧清单；App Store 当前强制边界。

### 2.2 A-02　iOS 缺 `NSPhotoLibraryAddUsageDescription`

- **已验证事实**：`ios/Runner/Info.plist:73-74` 仅 `NSPhotoLibraryUsageDescription`，文案为「请允许APP保存图片到相册」；全文件无 Add 键。
- **原始描述的第二个断言（功能受影响）的反证**：
  1. `lib/utils/image_utils.dart:85-88`：`requestPer()` 在非 Android 时执行 `Permission.photos.request()`。`permission_handler_apple` 的 `Permission.photos` 对应 `PHPhotoLibrary.requestAuthorization(for: .readWrite)`，其所需键是 `NSPhotoLibraryUsageDescription` —— **本仓已声明**。
  2. 插件写入路径 `SaverGalleryPlugin.swift:246-255` 直接 `PHPhotoLibrary.shared().performChanges`，不请求 add-only 授权。
  3. Apple 官方文档（Photos 隐私指南）：「If your app only adds to the library, use the NSPhotoLibraryAddUsageDescription key. **For all other cases, use NSPhotoLibraryUsageDescription**」，并给出 read/write 授权后 `performChanges` 保存的官方示例 —— 即 read/write 授权下写入成立。
  4. 公开经验证据（Stack Overflow 同题）：已通过 `PHPhotoLibrary` 取得 read/write 授权后，写入不再需要 Add 键；崩溃/失败报告集中在「尚未授权 + 走 add-only 级 API（如系统「存储图像」菜单）」。
- **残余风险路径（不排除）**：`pages/login/view.dart:84` 与 `pl_player/controller.dart:3288` 直接调用 `saveByteImg` 未先请求权限，首次保存时由 `performChanges` 触发隐式授权，此时是否需要 Add 键无法从源码确定。
- **最小验证方式**：iOS 真机，两个用例分别执行 ——（i）先在「分享面板→保存图片」路径触发一次授权后再保存（`save_panel/view.dart:289→319`）；（ii）卸载重装后直接走登录页二维码保存（`login/view.dart:84`）。观察是否弹窗、是否返回 `isSuccess=false`、是否 SIGABRT。
- **影响与严重程度**：合规/审核语义层面成立（写入场景未配 Add 键、现有文案是写入语义）；「功能已损坏」结论不成立或至少证据不足。
- **缺少信息**：真机两用例结果；`ios/Pods` 生成后插件的授权前置逻辑是否另有封装。

### 2.3 A-03　Bundle ID / `DEVELOPMENT_TEAM`

- **已验证事实**：`project.pbxproj:395/525/549` = `PRODUCT_BUNDLE_IDENTIFIER = com.PiliMax.android`；`:388/518/542` = `DEVELOPMENT_TEAM = ""`（报告行号逐条命中）。
- **反证（决定性）**：`.github/workflows/ios.yml:40` 使用 `flutter build ios --release --no-codesign`，`:46-49` 对 `Payload/Runner.app/Frameworks` 逐个 `codesign --force --sign - --preserve-metadata=identifier,entitlements`，并注明 "make AltSign happy"，最终产出 `PiliMax_ios_*.ipa`。即：**iOS 交付物是不签名 / ad-hoc 重签的侧载 IPA，不存在 Team 签名环节**。
- **技术事实澄清**：`com.PiliMax.android` 作为反向 DNS 标识本身合法（Apple 只要求反向 DNS 形式与唯一性），「后缀误用」属命名观感问题而非上架拒绝项。
- **最小验证方式**：`grep -n "DEVELOPMENT_TEAM" ios/Runner.xcodeproj/project.pbxproj` 与 `grep -n "PRODUCT_BUNDLE_IDENTIFIER" ...`，再对照 `ios.yml` 构建命令。
- **影响与严重程度**：在真实分发路径下为「无影响」；报告「影响：签名与上架」的前提不成立。
- **缺少信息**：无。

### 2.4 A-04　Android 缺 `dataExtractionRules`

- **已验证事实**：`android/app/src/main/res/xml/` 目录**不存在**（`res/` 下仅有 drawable*/mipmap*/raw/values*/）；`AndroidManifest.xml:19-26` 的 `<application>` 属性为 `allowBackup="false"`、`fullBackupContent="false"`，无 `android:dataExtractionRules`；`build.gradle.kts:50` `targetSdk = 37`（远超 31）。
- **机制层面**：报告引用的官方条文（target 31+ 时 `allowBackup="false"` 只关闭云备份、不关闭 D2D）与 Android 12 行为变更文档的公开内容一致，但**本次环境无法访问该域名，未能独立复核原文**，故置信度记为「中」。
- **反证排查**：`fullBackupContent="false"` 只对 Android 11 及以下的完整备份生效，不能替代 `dataExtractionRules` 对 12+ 的 D2D 约束。
- **最小验证方式**：`adb shell bmgr list transports` / 用 Android 12+ 设备走「设备间数据迁移」流程，观察 App 私有目录（MMKV 文件、cookie 库）是否随迁移带出。
- **影响与严重程度**：隐私合规（中）；程度随厂商实现。
- **缺少信息**：官方条文原文的独立复核。

### 2.5 A-05　改字号后列表行高错乱（机制链验证）

逐环核验，**每一环均在工作区代码中成立**：

1. 保存路径：`lib/pages/setting/pages/font_setting.dart:137-146` —— 写入 `SettingBoxKey.defaultTextScale` 后 `Get..back()..updateMyAppTheme()..appUpdate()`（`:143-146` 与报告一致；另见 `:239-241`（删除字体）、`:280-282`（清空字体库））。
2. `appUpdate()` 语义：getx（`get` 为 git 依赖，`ref: dev`，本机缓存 commit `3e65da8…`，与报告引用一致）`extension_navigation.dart:516` = `void appUpdate() => _getxController.update();`；`_getxController` 为 `static GetMaterialController`（`:631`），`GetMaterialController extends SuperController`（`root_controller.dart:4`）。
3. `GetxController.update()` → `get_controllers.dart:16-21` → `refresh()`，即**仅通知监听者重建 widget**，不重建 `State`。
4. 对比反证（报告所述亦成立）：`updateLocale`（`:494-497`）走 `forceAppUpdate()`（`:512-514` → `engine.performReassemble()`）—— 那才是重建级操作。
5. 根节点确实会重建：`get_material_app.dart:191` `Widget build(...) => GetBuilder<GetMaterialController>(init: Get.rootController, ...)` → `refresh()` 会重建整棵 `MaterialApp`，`main.dart:464` 的 `TextScaler.linear(Pref.defaultTextScale)` 随之刷新。**因此文字会变大，而缓存的行高不会变**。
6. 缓存点实例：`lib/pages/rcmd/view.dart:57-63` 字段级 `late final gridDelegate = SliverGridDelegateWithExtentAndRatio(... mainAxisExtent: MediaQuery.textScalerOf(context).scale(90))`；同 State `:23-28` 混入 `AutomaticKeepAliveClientMixin` 且 `wantKeepAlive => true` → State 长期存活，`late final` 只求值一次。

**对报告证据的一处修正**：报告中「约 30 处 delegate 在字段初始化时读取 `MediaQuery.textScalerOf(context)` 并存入 `late final`」**计数不准确**。实测全仓 `textScalerOf` 命中约 30 处，但其中**字段级 `late final` 仅 14 处**（`fav/topic:56`、`live_follow:56`、`live_search/child:78`、`live_area_detail/child:71`、`member_pgc:70`、`member_like_arc:73`、`member_coin_arc:73`、`member_home:49,59`、`pgc:316`、`rcmd:57`、`pgc_index:226`、`search_panel/live:48`），另有 3 处是**方法内局部** `late final`（`search/view.dart:289`、`save_panel/view.dart:340`、`member_upower_rank/view.dart:181`，每次 build 重新求值、不产生陈旧值）。机制结论不受影响（rcmd 正是字段级 + keepAlive 的样例），但影响面统计需按字段级重算。

- **最小验证方式**：真机 release 包 →「设置→App字体设置」把字号拉到 1.6 → 返回首页（rcmd）/ 搜索 / UP 主页；观察是否出现文字裁切、`RenderFlex overflowed` 黄黑条纹（debug 包可直接看到）。另一对照：完全杀掉进程后重进同一页，判断是否消失。
- **严重程度**：中（视觉缺陷，非崩溃）。
- **缺少信息**：真机视觉确认；影响页面的完整清单。

### 2.6 A-06　`Grid.smallCardWidth` 为 `static final`

- **已验证事实**：`lib/utils/grid.dart:27` `static final double smallCardWidth = Pref.smallCardWidth;`；`storage_pref.dart:550-551` 从设置盒读取（默认 240.0）；写入点全仓唯一（`style_settings.dart:858`），无重置入口；`style_settings.dart:855-861` 保存后 `SmartDialog.showToast('重启生效')`（`:860`）—— 与报告行号一致。
- **对报告措辞的修正**：报告称「被 40+ 处 grid delegate 引用」。实测 `Grid.smallCardWidth` 共 41 行命中，但其中 `pages/article/view.dart:75`、`pages/dynamics_detail/view.dart:492`、`pages/music/view.dart:115`、`pgc/view.dart:71,178,364,435` 等属**内边距/宽度换算**，并非 grid delegate 参数；严格意义上「grid delegate 引用」不足 40 处。
- **最小验证方式**：设置「列表最大列宽度」改值 → 不重启，观察首页/其他列表列数是否变化（预期不变）；杀进程重进则生效。
- **严重程度**：低（设置不即时生效，不造成错乱）。
- **结论口径建议**：与 `style_settings.dart:860` 的自我提示结合看，此项更接近「设计如此/已知限制」而非未见缺陷。本次仍按「事实真实存在」记录，同时标注定性存疑。

### 2.7 A-07　`CFBundleURLTypes` 结构嵌套错误

- **已验证事实（逐行）**：`ios/Runner/Info.plist:85-127` 外层 array；`:87-117` 第一个 dict；其中 `:95` 又出现 `CFBundleURLTypes`，其内 array（`:96-116`）含 `:97-115` 一个 dict、12 个域名（`:102-113`）；`:92-93` 为 `http`/`https`；`:119-126` 为第二个 dict（`bilibili` scheme）。
- **反证核对**：报告对该项的自我修正（未知键被忽略而非解析异常）成立 —— `CFBundleURLSchemes`/`CFBundleURLName` 是系统读取键，嵌套的同名键不会被递归解析，故后果是 12 条域名条目失效，而非 schema 异常。
- **最小验证方式**：`plutil -lint ios/Runner/Info.plist`（语法合法）+ `plutil -p` 输出对照 `UNUserNotificationCenter`/`openURL` 实际可用 scheme 列表。
- **严重程度**：低（不影响既有 `bilibili` scheme 与 `http(s)` 处理）。

### 2.8 A-08　关闭弹幕后弹幕层仍逐帧重绘

- **已验证事实**：
  - 本仓 `lib/pages/danmaku/view.dart:226-231` `build` 返回 `Obx(() => AnimatedOpacity(opacity: enableShowDanmaku.value ? danmakuOpacity.value : 0, ... child: DanmakuScreen(...)))` —— 关闭时仅把 opacity 置 0，`DanmakuScreen` 仍挂载（行号与报告一致）。
  - `:90-98` `playerListener(PlayerStatus)` 是唯一的 resume/pause 调用点（`:93`/`:95`），由播放状态驱动。
  - `:101-104` `videoPositionListen` 在 `!enableShowDanmaku.value` 时提前 return → 数据管线确实停止（与报告的严重程度修正一致）。
  - 插件（`canvas_danmaku-3947b1f…`，与报告引用 commit 一致）：`:71` `bool _running = true;`；`:282` 是 `_pause()` 内唯一置 false 处（另 `:150` 在 dispose）；`:526`、`:574` 两处 `CustomPaint(willChange: _running)`。
  - `lib/plugin/pl_player/view/view.dart:341-344` `enableShowDanmaku.listen` 回调仅 `_removeDmAction()` 与切换 `onTapDown`，**不调用 pause**（与报告一致，且该文件该区段在工作区与快照仓一致）。
- **结论**：原文所述「关闭弹幕后仍逐帧重绘」在代码层面成立（ticker 的 `_running` 保持 true，弹幕层保持已注册的 `willChange` 层）。
- **最小验证方式**：真机 release/ profile → 播放视频并开启弹幕 → 关闭弹幕开关 → 用 DevTools Performance 对比 raster/UI 线程帧耗时与 `Opacity` 层是否存在持续重绘。
- **严重程度**：低-中（量级未实测）。**缺少信息**：每帧实际成本的 profile 数据。

### 2.9 A-09　播放器表面探针稳定后不再停止

- **已验证事实（工作区）**：
  - `lib/plugin/pl_player/view/view.dart:2900`（`didUpdateWidget` 的 `viewportSize` 分支）：`_resetStability(notify: false, preserveSurface: true);`（该分支 `:2896-2902`）。
  - `:2999-3013` `_resetStability`：`:3005` **无条件** `_startProbeTimer()`；`:3006` `if (!_surfaceReady || preserveSurface) return;` → `preserveSurface == true` 时提前返回，**保留 `_surfaceReady = true`**。
  - `:3121-3126` 停止分支：`final requiredStableFrames = _hasShownSurface ? _stableFrameCount : 1;` / `if (_stableFrames >= requiredStableFrames && !_surfaceReady) { _hasShownSurface = true; setState(() => _surfaceReady = true); _stopProbeTimer(); }`。由于上一步 `_surfaceReady` 恒为 true，`!_surfaceReady` 恒假 → **停止分支不可达**，100ms `Timer.periodic`（`:2959-2967`）持续运行。
  - `:3099`：几何未就绪分支传 `_resetStability(preserveSurface: _hasShownSurface)`，`_hasShownSurface` 在就绪时（`:3123`）置 true —— 与报告描述一致。
  - 边界（反证）：`:2994` `_detachController()` 内 `_stopProbeTimer()`；调用点 `:2906`（`_attachController`）与 `:3140`（`dispose`）；`:2913` `_resetStability(notify: false)` 使用默认 `preserveSurface: false` → `_surfaceReady` 复位 → 停止分支重新可达。故缺陷窗口限于「同一控制器 + 视口变化后」至控制器切换/销毁之间。报告的边界分析成立。
- **与报告的差异（必须说明）**：报告引用的 `:3103-3107`、`:2981-2995`、`:2956-2979`、`:3081`、`:3104` 对应**快照仓**行号；工作区同段代码整体后移约 15-18 行且停止条件重构（见 0.2）。**判定结论在两侧均成立**（工作区 `_hasShownSurface` 为 true 时 `requiredStableFrames = 3`，与快照仓 `_stableFrameCount` 等价）。
- **最小验证方式**：真机（或 profile 模式）进入视频详情 → 旋转/全屏/进入退出小窗触发 `viewportSize` 变化 → 在 `_startProbeTimer`/`_stopProbeTimer` 打日志或断点，观察计时器是否在画面稳定后仍未停止；对照：切换一次视频源后是否停止。
- **严重程度**：中（持续 100ms 周期唤醒 + 每帧 `addPostFrameCallback` + async `endOfFrame` 等待，影响耗电与帧预算）。
- **缺少信息**：真机 profile 下的实际耗电/帧耗时增量。

### 2.10 A-10　`clickedBvids` 只增不减

- **已验证事实**：活路径 `lib/pilimax/forks/common/widgets/video_card/video_card_h.dart:23` `static final RxSet<String> clickedBvids = <String>{}.obs;`；`:180` `VideoCardH.clickedBvids.add(key);`；`:403` `VideoCardH.clickedBvids.contains(key)`（用于标题变灰 `:409`）。全仓 grep 仅此 3 处，无 remove/clear。快照仓完全相同。
- **反证**：`:403` 的消费语义是「本会话内已点开过的视频标题变灰」，属会话内标记，无 clear 反而是其设计需要（进程内即失效）。
- **最小验证方式**：连续浏览 500+ 卡片，观察 `clickedBvids.length` 单调增长（需 debug 观察点）与内存占用（字符串集合，KB 级）。
- **严重程度**：低。**缺少信息**：产品意图确认。

### 2.11 A-11　`DanmakuMergeWorkerClient` 无实例化点

- **已验证事实**：全仓 grep `DanmakuMergeWorkerClient` 仅命中 `lib/pilimax/utils/danmaku_merge/worker_client.dart:14`（类）、`:15`（构造函数）—— 与报告一致。
- **加强证据（报告未给出）**：`lib/pilimax/utils/danmaku_merge/` 整目录 8 个文件的外部引用为零：所有 import 均指向目录内（`worker_client.dart:8-11`、`worker_entry.dart:7-12`、`clusterer.dart:7-10`、`similarity_matcher.dart:7-8`、`worker_models.dart:6`）。即被隔离的是**整个 isolate 化合并模块**，而非单个 client。
- **对照（说明合并功能并未缺失）**：实际合并走主隔离区路径 —— `lib/pages/danmaku/controller.dart:19` `_mergeDanmaku = _plPlayerController.mergeDanmaku`、`:105-113` 按内容去重计数。pubspec 已声明 `assets/danmaku_merge/pinyin_dict.txt`（`pubspec.yaml:291`），倾向判定为**在建/未接线代码**而非遗漏删除。
- **最小验证方式**：`grep -rn "worker_client.dart" lib/` 零命中即为证据；若要确认无动态加载，检索 `Isolate.spawn` 的调用点。
- **严重程度**：无（无运行时行为；仅可维护性与包体影响）。

### 2.12 A-12　`res/raw/keep.xml` 的 `tools:keep="@drawable/*"`

- **已验证事实**：`android/app/src/main/res/raw/keep.xml:2-3` 内容为 `<resources xmlns:tools=... tools:keep="@drawable/*" />`；`android/app/build.gradle.kts:94-97` 的 `proguardFiles(...)` 处于注释状态，且文件内无 `isShrinkResources`/`shrinkResources` 配置（AGP 默认 false）。
- **反证核算**：`keep.xml` 仅在被资源压缩器读取时才生效；当前未启用资源压缩，故该规则不产生任何构建影响。
- **最小验证方式**：`./gradlew :app:assembleRelease` 后解包 APK 检查 `res/` 中 drawable 是否被裁剪；或在 `build.gradle.kts` 临时开启 `isShrinkResources` 做对照（属验证手段，不构成修复建议）。
- **严重程度**：当前无；一旦开启资源压缩，`@drawable/*` 通配会保留全部 drawable（与「保留少量入口」的常规意图相反）。

### 2.13 X-01　两套连接池（误报）

- **已验证事实（import 图逐条核验）**：
  - legacy `lib/http/init.dart` 的 import 方恰为 3 个：`lib/utils/accounts.dart:1`、`lib/services/download/download_manager.dart:4`、`lib/utils/wbi_sign.dart:9`。
  - legacy `lib/utils/accounts.dart` 的 import 方恰为 4 个：`lib/http/init.dart:10`、`lib/utils/accounts/account.dart:3`、`lib/utils/storage.dart:7`、`lib/pages/later/controller.dart:13` —— **全部位于 legacy 簇内部**。
  - legacy `lib/pages/later/controller.dart` 的唯一 import 方是 `lib/pages/later/widgets/video_card_h_later.dart:12`；而 `lib/pages/later/view.dart:13`、`child_view.dart:8`、`later_search/controller.dart:7` 引用的是 **forks** 版本；`lib/pages/later/widgets/video_card_h_later.dart`（legacy）**零引用方**。
  - legacy `lib/services/download/download_manager.dart`、`lib/utils/wbi_sign.dart` 亦零外部引用（`services/download/download_service.dart:23` 用 forks 版）。
- **结论**：legacy 簇为**闭合且无 live 入口**的子图，其静态 `dio`/`_http11Dio` 不会被初始化 → 「运行期双连接池」不成立。报告对本项的反证与自我更正（含「唯一 live 入口是 later/controller.dart:13」这一旧论据的撤回）**均与实测一致**。
- **补充确认**：`tool/pilimax/fork_map.tsv:8` 登记 `fork lib/http/init.dart lib/pilimax/forks/http/init.dart`，与「有意保留的上游快照」定性一致。
- **最小验证方式**：在 legacy `init.dart` 的静态初始化/构造中加断点或日志，冷启动走主要路径，观察是否命中。

### 2.14 X-02　`WRITE_SETTINGS` 过度声明（误报）

- **已验证事实**：`lib/plugin/pl_player/view/view.dart:388` 调 `ScreenBrightnessPlatform.instance.setSystemScreenBrightness(...)`；插件 `screen_brightness_android-2.1.6` 的 `ScreenBrightnessAndroidPlugin.kt:139` 分发 `setSystemScreenBrightness` → `:172` 调 `setSystemScreenBrightness(context, brightness)` → `:335` 定义 → `:336-346` 内 `if (!canWriteSystemSetting(context)) { ... Settings.ACTION_MANAGE_WRITE_SETTINGS ... return false }`；`canWriteSystemSetting` `:382-388` → `:384` `Settings.System.canWrite(context)`。
- **结论**：`WRITE_SETTINGS` 是系统级亮度调节的必要权限（否则会触发跳设置页），本项「过度声明」不成立。报告行号（`:335-351`、`:384`）与实测一致。

### 2.15 X-03　`READ_MEDIA_AUDIO/VIDEO` 弹窗（误报）

- **已验证事实**：全仓权限请求仅 `Permission.storage.request()`（`image_utils.dart:87`，Android）、`Permission.photos.request()`（`:88`，iOS 或 Android 13+ 图片）；`Permission.audio.request`、`Permission.videos.request` 零命中。
- **结论**：仅声明不请求 → 不会弹窗。原描述不成立；两项权限属清单冗余（对商店数据安全表单有影响）。
- **边界说明**：`READ_MEDIA_IMAGES` 确有对应请求路径（`Permission.photos`），与本次判定无关。

### 2.16 R-10 / M-14（§6.2，误报成立）

- **R-10**：`lib/common/widgets/flutter/vertical_tabs.dart`（fork 源，非活路径）= 2404 行 / 2172 非空行；活路径 `lib/pilimax/forks/common/widgets/flutter/vertical_tabs.dart` = 218 行 / **204 非空行**，`Widget build` 仅 `:21`、`:176`。故「build 约 1605 行」不成立（属旧路径），报告判定**正确**。
- **M-14**：活路径无 `GlobalKey` 生成逻辑 —— `VerticalTab` 为 `StatelessWidget`（`:3-46`），`_VerticalTabBarState`（`:94-118`）以 `TabController.animation` 监听 + `setState` 驱动，不产生每 tab 的 key 重建。报告判定**正确**。

### 2.17 L-06（§6.2，报告的「误报」判定不成立）

- **核查过程**：回到原始条目 `mobile_audit_full.md:228`，其位置列明确写的是 `pilimax/forks/common/widgets/flutter/vertical_tabs.dart:183` —— **即活路径**，而非报告 §6.2 表格中所写的旧路径。同时该条目还列出 `member/widget/medal_wall.dart:26,67`、`dynamics/widgets/vote.dart:90,94,103`、`audio/view.dart:983,985`、`setting/models/style_settings.dart:481` 等多个**活文件**。
- **活文件复核**：`pilimax/forks/.../vertical_tabs.dart` 中 `shrinkWrap: true` 位于 `:186`（报告写「:183」，实际 `:183` 是 `? ListView(`，偏移 3 行）；`pages/dynamics/widgets/vote.dart` 中 `shrinkWrap: true` 位于 `:91`、`:104`（原条目写 `:90,94,103`，偏移 1 行）。
- **结论**：该条**不是**「指向已废弃 fork 源」的问题，而是「行号轻微偏移 + 模式在活代码中确实存在」的问题。报告把它归入「已确认为误报」并计入 6 项误报，属**依据错误**；正确处置应为「回炉重判」。
- **最小验证方式**：`grep -n "shrinkWrap" lib/pilimax/forks/common/widgets/flutter/vertical_tabs.dart lib/pages/dynamics/widgets/vote.dart`。

### 2.18 D-01 ~ D-04（设计如此，均成立）

- **D-01**：本仓 `AndroidManifest.xml:174-182`（AudioService，含 `exported="true"`、`foregroundServiceType="mediaPlayback"`、`tools:ignore="Instantiatable"`、`MediaBrowserService` intent-filter）与 `:188-195`（MediaButtonReceiver）；上游 example（`Pub\Cache\git\audio_service-e0860cf…\audio_service\example\android\app\src\main\AndroidManifest.xml:39-47, 49-56`）声明内容一致。结论成立；仅「逐行一致」措辞偏强。
- **D-02**：`lib/pages/video/controller.dart:2517-2522`（`onClose` 中 `if (isEnteringPip) return;`，报告写 `:2518-2522`，与 `void onClose() {` 在 `:2518` 一致）；`lib/pilimax/services/pip_overlay_service.dart:124-130` 注释、`:131-149` `_releaseSavedVideoOwner`（`:137` 置 `isEnteringPip = false`；`:142-147` 按 `disposePlayer` 决定 `dispose()`/`pause()`）。结论成立。
- **D-03**：`lib/main.dart:464`、`:487-508`（两个分支均传 `textScaler`）、`storage_pref.dart:1357-1358`（默认 1.0）、全仓「跟随系统」唯一命中 `models/common/theme/theme_type.dart:8`（主题模式选项）。结论成立。
- **D-04**：`tool/pilimax/fork_map.tsv:4`。结论成立。

### 2.19 U-01　iOS Multicast entitlement（本次**升级**为真实存在）

- **已验证事实**：
  - `ios/Runner/Release.entitlements` 与 `DebugProfile.entitlements` 内容均仅为空的 `keychain-access-groups` 数组，无 `com.apple.developer.networking.multicast`。
  - Apple 官方 entitlement 文档（`developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.networking.multicast`）原文：「Your app must have this entitlement to send or receive IP multicast or broadcast on iOS.」（iOS 14.0+；且该 entitlement 需向 Apple 申请）。
  - 可达性：`lib/router/app_pages.dart:200` 无条件注册 `GetPage(name: '/dlna', ...)`；`lib/pages/video/controller.dart:2975` `Get.toNamed('/dlna', ...)`；`lib/pages/live_room/controller.dart:711` `'/dlna',`；`lib/pages/dlna/view.dart` 的 import 列表（`:1-8`）不含 `dart:io`/平台判断工具 → **无平台门控**。三条路径与报告引用完全一致。
- **为何仍留「中」置信度**：`dlna_dart` 源码不在本机缓存，其 SSDP 是否使用组播 socket 未在本地核实（DLNA/SSDP 依赖 239.255.255.250:1900 组播为行业共识，但本报告不将其当作已验证事实）；且模拟器不复现组播行为。
- **补充观察**：本仓 iOS 交付为侧载未签名 IPA（3.1），而 multicast entitlement 需 Apple 团队资格审批 —— 即使代码侧补齐清单，该能力在当前分发模型下也难以获得。
- **最小验证方式**：iOS 真机 + 同一局域网 + 可投屏设备 → 进入任意视频页「投屏」→ 观察设备列表是否为空；对照 Android 同网环境。
- **严重程度**：中（iOS 端投屏功能不可用）。

### 2.20 U-02　`network_security_config.xml`（本次**改判**为误报倾向）

- **已验证事实**：`android/app/src/main/res/xml/` 目录不存在；`AndroidManifest.xml` 无 `android:networkSecurityConfig`；`targetSdk = 37`。
- **决定性反证**：Flutter 官方破坏性变更文档（network policy for iOS/Android）明确写道：「**Important** The following only applies to platform native sockets (sockets owned by the Android and iOS platforms). **Flutter does not enforce any policy at socket level**; you would be responsible for securing the connection. **If the socket is owned by Dart/Flutter, no policy will be enforced.**」；flutter/flutter#106678 的复现与讨论亦证实 `dart:io` 的 `HttpClient` 在 `usesCleartextTraffic="false"` 下仍可发 http 请求。
- **对本仓的映射**：HTTP 栈为 Dio（`pubspec.yaml:84`）→ `dart:io HttpClient`（`lib/http/init.dart` forks 版使用 Dio/`dio_http2_adapter`）；DLNA 走 Dart socket。二者均属「Dart 自有 socket」，不受 Android `NetworkSecurityPolicy` 约束。
- **残余不确定（不排除）**：WebView（`flutter_inappwebview`）加载 http 页面、原生播放器（media-kit/mpv）内嵌 http 资源这两条由平台/原生库实施的路径可能受策略影响；本环境无法实测。
- **最小验证方式**：（i）真机 release 包中调用一次 http 地址的 Dart 请求，观察是否抛 `CLEARTEXT communication ... not permitted`；（ii）WebView 加载 `http://` 页面对照。
- **严重程度**：低（原描述「cleartext 请求失败」在主链路上不成立）。

---

## 3. 与报告结论的差异汇总（本次校验发现的问题）

### 3.1 报告隐含的「上架分发」前提与实际分发模型冲突
报告 §2.1「上架与合规类（4 项）」的影响判定，均建立在「App Store 上架」这一前提上。但 `lib/scripts/build.ps1` + `.github/workflows/ios.yml:37-49` 显示 iOS 交付为 `--no-codesign` 构建后 ad-hoc 重签、打包 `PiliMax_ios_*.ipa` 的**侧载分发**流程。这直接改写了 A-01、A-02（合规部分）、A-03 的影响判定：在这条分发路径下，它们不构成阻塞。
（说明：这是对「影响」的修正，不改变 A-01/A-02 配置项确实缺失这一事实。）

### 3.2 版本基线未标注
报告未声明其行号对应哪一份快照。实测 `lib/plugin/pl_player/view/view.dart` 在工作区与快照仓存在差异（`requiredStableFrames` 重构），若读者按工作区回查会得到错误位置。A-09 的结论虽仍成立，但**行号级引用必须标注基线**。

### 3.3 报告自身的一处误判（L-06）
§6.2 把 L-06 记为「引用路径指向已废弃 fork 源 → 误报」，但原始条目的位置列写的是**活路径** `pilimax/forks/.../vertical_tabs.dart:183`，且 `shrinkWrap: true` 在活文件中确实存在（`:186`）。该项的「误报」结论**依据不成立**，应回到待重查。

### 3.4 编号体系不自洽
§1.2 计数表写「已确认为误报 6 项，编号 X-01 ~ X-06」，但 §三 只定义了 X-01 ~ X-03；另 3 项误报以 R-10/M-14/L-06 之名出现在 §6.2，而这三者在原始清单（`mobile_audit_full.md`）中属「渲染项/回归项」编号段。同一份报告内存在两套编号指向同一组条目，易造成引用混乱。

### 3.5 计数类表述需收紧
- A-05「约 30 处 delegate 在字段初始化时存入 `late final`」→ 实测字段级 14 处，另 3 处为方法内局部（不受影响）。
- A-06「被 40+ 处 grid delegate 引用」→ 41 行命中中含非 grid delegate 的边距/宽度换算用途。
- D-01「逐行一致」→ 语义一致，字面不一致（属性顺序与条目位置不同）。

---

## 4. 总结

### 4.1 计数（本次校验口径，共 24 项 + 3 项 §6.2 条目）

| 结论 | 数量 | 条目 |
|---|---|---|
| **真实存在** | **13** | A-01、A-02（配置项）、A-03（代码事实）、A-04、A-05、A-06（事实）、A-07、A-08、A-09、A-10、A-11、A-12、U-01 |
| **误报** | **6** | X-01、X-02、X-03、R-10、M-14、U-02（由「信息不足」改判为误报倾向） |
| **设计如此** | **4** | D-01、D-02、D-03、D-04（A-06 的定性亦可归入此类，但代码事实成立，故计入「真实存在」列并标注） |
| **报告判定不成立 / 待重判** | **1** | L-06（报告判「误报」，实际依据错误） |
| **影响表述需修正** | 3 | A-01（上架前提）、A-02（功能影响）、A-03（签名与上架影响） |
| **报告内部编号不一致** | 1 处 | §1.2 的 X-01~X-06 vs §三/§6.2 |

对照报告自报数（真 12 / 误报 6 / 设计如此 4 / 信息不足 2）：
- 报告 12 项「真」中，A-06 的定性偏「设计如此」，但代码事实成立，故计数不变；**A-01/A-02/A-03 的影响层面需下调**（配置事实仍成立）。
- 报告 2 项「信息不足」中，**U-01 可升级为真实存在（机制成立，有 Apple 官方条文）**，**U-02 可改判为误报倾向（有 Flutter 官方条文）**。
- 报告 6 项「误报」中，**L-06 的依据不成立**，实际应为 5 项成立 + 1 项待重判。

### 4.2 复盘：是否存在误判（对本报告的自我检查）

1. **A-02 的判定方向是否过强？** 我以「应用在 iOS 走 `Permission.photos`（readWrite）」+「Apple 官方 read/write + performChanges 示例」+「SO 同题经验」论证功能未坏。最强反例是「未预先授权 + `performChanges` 隐式授权」路径（`login/view.dart:84`、`pl_player/controller.dart:3288`），该路径我无法在源码层面判定 iOS 使用哪个键，因此我把功能影响的置信度压到「低」并明确列为待真机验证 —— 若真机在该路径出现 SIGABRT，本判定应改为「真实存在（含功能影响）」。**这是本报告最需要外部验证的结论。**
2. **A-06 / A-10 / A-11 是否应归入「真实存在」？** 三者代码事实无疑，但性质分别是「已知限制」「可能的会话标记」「未接线代码」。若按缺陷口径统计，存在低估风险；我已逐条标注定性，并在总表保留「真实存在（事实）」。计数口径若改为「按缺陷定性」，真实存在应由 13 降为 10。
3. **A-09 的行号漂移是否影响结论？** 我在工作区重新推导了完整可达性链（`:2999-3007` → `:3121-3126`），并非照搬报告结论；两侧均成立。若后续工作区再次改动 `requiredStableFrames` 语义（例如让首次显示也要求 3 帧或让 `_surfaceReady` 参与该表达式），结论需重算。
4. **X-01 的结论是否依赖「无动态引用」假设？** 是。我核验的是静态 import 图；若存在 `dynamic`/反射式调用（Dart 无反射式实例化，故风险极低），结论需修正。这也是我未把 X-01 标为「绝对不可能」的原因。
5. **U-02 的改判是否过度？** 我引用的 Flutter 官方条文只覆盖「Dart/Flutter 自有 socket」，未覆盖 WebView 与原生播放器内嵌 http。我已在结论中限定残余不确定范围，并把置信度定为「中高」而非「高」。
6. **是否有条目因环境限制而被高估/低估？** 两处：A-04（官方条文域名不可达，置信度降为中）、U-01（`dlna_dart` 源码不可得，置信度保持中）。二者均未以猜测填补。

### 4.3 待补充信息清单（按影响排序）

| 序号 | 缺什么 | 影响的条目 |
|---|---|---|
| 1 | iOS 真机 + "首次保存（未预授权路径）"实测 | A-02（决定「功能影响」成立与否） |
| 2 | 真机验证「改字号 → 返回列表」是否视觉裁切 | A-05 |
| 3 | 真机 profile（帧耗时/耗电）复核 100ms 探针与弹幕层开销 | A-09、A-08 |
| 4 | iOS 真机 + 局域网 + 可投屏设备实测 SSDP | U-01 |
| 5 | Android 12 行为变更文档原文核对 | A-04 |
| 6 | 生成 `ios/Pods` 后核对 Pod/SPM 侧隐私清单；App Store 对宿主清单的强制边界 | A-01 |
| 7 | 构建产物级验证：`--no-codesign` 构建后检查 App 包内 cleartext/清单/entitlement 实际状态 | A-01、A-04、U-01、U-02 |
| 8 | 产品意图确认（`clickedBvids` 是否为会话标记；`danmaku_merge` 是否在建） | A-10、A-11 |
| 9 | 是否在原 88 项清单中重判 `shrinkWrap` 类条目 | L-06 及其同条目其余位置 |

---

*本报告为只读校验产物，未对任何源码文件执行写入、重命名或删除操作；未提供修复方案、patch 或 diff。*
