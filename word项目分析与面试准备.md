# Unity 项目面试分析文档

## 项目简介

这是一个 Unity 2022.3 移动端英文单词连线/填字休闲游戏，主平台以 Android 为主。核心玩法类似 Wordscapes：玩家在圆盘字母区滑动连词，填充上方 crossword 网格，同时包含额外词、词典解释、关卡推进、道具提示、登录奖励、商店、去广告、礼包、广告变现、Firebase 埋点和远程配置等商业化能力。

关键入口与模块：

- `Assets/AppEngine/Runtime/GameEntrance.cs`：框架启动入口。
- `Assets/Scripts/Game/WordGameFlowBridge.cs`：业务流程桥接。
- `Assets/Scripts/Game/WordModule/WordMainUI.cs`：玩法主控。
- `Assets/Scripts/Game/WordModule/WordGuess/WordGuessUI.cs`：连线拼词输入与道具。
- `Assets/Scripts/Data/LevelData.cs`：关卡运行数据。
- `Assets/Editor/BuildScript.cs`：Android APK/AAB/Gradle 工程构建入口。

## 技术栈

- 引擎与语言：Unity 2022.3、C#、UGUI、Coroutine、PlayerPrefs。
- UI 与动画：UGUI、DOTween、ParticleSystem、UI Particle。
- 数据与资源：JsonUtility、Newtonsoft.Json、ScriptableObject、CSV/JSON 打表、AssetBundle、StreamingAssets。
- Android：`AndroidJavaClass`、`AndroidJavaObject`、Gradle 模板、Manifest、Proguard 配置。
- 商业化：广告 SDK、Unity IAP、去广告、金币/道具商店、礼包、激励视频。
- 运营与分析：Firebase Analytics、Remote Config、ABTesting、Adjust、Google Play Review。
- 工具链：Unity EditorWindow、关卡生成器、词典导入、配置表导出、APK/AAB 构建脚本。

## 整体架构

### 启动流程

项目不是把初始化堆在一个 `Start` 中，而是通过 `GameEntrance -> App -> ProcedureFSM` 拆分启动阶段：

- `ProcedureLaunch`：启动加载阶段。
- `ProcedurePreload`：预加载词典、关卡、配置数据。
- `ProcedureInitSDK`：初始化或同步广告、Remote Config 等 SDK 状态。
- `ProcedureStartGame`：进入首页或游戏业务。

业务通过 `IAppFlowBridge` 与框架解耦，`WordGameFlowBridge` 负责：

- 预加载词典：`DicManager.Instance.LoadDic`
- 进入首页：切换 `MainScene` 并打开 `MainUI`
- 进入玩法：切换 `GameScene` 并打开 `WordMainUI`

### UI 架构

`UIManager` 管理 Main、Pop、Guide、Top 多层 Canvas，支持：

- UI 缓存与预加载。
- 参数化打开 UI。
- 弹窗层级隔离。
- 安全区适配。
- Pad 比例适配。
- 引导层独立显示。

`UIBase` 统一 Show、Hide、Close 生命周期，并通过内部 `GameEventMgr` 在关闭或隐藏时清理事件，降低 UI 关闭后仍收到事件导致的异常风险。

### 玩法架构

玩法主链路：

1. `WordGuessUI` 处理触摸/鼠标连线输入。
2. 拼词结束后发送 `OnSpellEnd`。
3. `WordMainUI` 判断重复词、关卡内新词、额外词、错误词。
4. `LevelData` 更新已拼词、已打开格子、额外词和关卡状态。
5. `WordsShowUI` 负责棋盘字母显示、翻开和表现。
6. `WordMainUI` 检查过关、播放结算和过场动画。

## 核心模块

### 1. 启动与流程模块

相关文件：

- `Assets/AppEngine/Runtime/GameEntrance.cs`
- `Assets/AppEngine/Module/ProcedureModule/ProcedureFSM.cs`
- `Assets/AppEngine/Module/ProcedureModule/Procudures`
- `Assets/Scripts/Game/WordGameFlowBridge.cs`

价值：

- 启动流程清晰。
- 资源加载、SDK 初始化、进入业务互相隔离。
- 后续接入新 SDK 或新增启动阶段成本较低。

### 2. 玩法模块

相关文件：

- `Assets/Scripts/Data/LevelData.cs`
- `Assets/Scripts/Game/WordModule/WordMainUI.cs`
- `Assets/Scripts/Game/WordModule/WordGuess/WordGuessUI.cs`
- `Assets/Scripts/Game/WordModule/WordsShow/WordsShowUI.cs`

价值：

- 支持连线拼词、重复词、新词、额外词、错误词判定。
- 支持 Hint、Target、Rocket 等关内道具。
- 支持关卡恢复、过关判断、结算、引导、飞字动画。
- 处理了道具动画 pending 状态，避免重复开格和重复扣费。

### 3. 数据与存档模块

相关文件：

- `Assets/AppEngine/Module/DataModule/RecordDataManager.cs`
- `Assets/AppEngine/Module/DataModule/DicManager.cs`
- `Assets/Scripts/DataVO`

价值：

- `RecordDataManager` 统一管理金币、关卡、道具、礼包、广告、引导、通知等存档。
- 本地存档有轻量加密，并保留旧 PlayerPrefs 数据 fallback。
- `DicManager` 统一加载词典、关卡、解释表、Level_1 数据源。

### 4. 资源与打表模块

相关文件：

- `Assets/Editor/EditorWindow/ExportTable/ExportTableWindow.cs`
- `Assets/Editor/EditorWindow/LevelBuilder/LevelBuilderWindow.cs`
- `Assets/Editor/EditorWindow/WordscapesImport/WordscapesLevelImporter.cs`
- `Assets/Editor/EditorWindow/ToolsEditor.cs`

价值：

- 支持 CSV 打表生成 C# VO 和 ScriptableObject。
- 支持 AssetBundle 导出。
- 支持词典、关卡、每日挑战数据打包。
- 支持外部 Wordscapes JSON 导入为项目关卡格式。
- 提升策划内容生产效率，减少人工配置错误。

### 5. SDK 与商业化模块

相关文件：

- `Assets/Scripts/Module/SDKModule/AdAndroidUtil.cs`
- `Assets/Scripts/Module/SDKModule/FirebaseUtil.cs`
- `Assets/Scripts/Module/SDKModule/RemoteConfigUtil.cs`
- `Assets/Scripts/Module/SDKModule/ReceiveJavaCall.cs`
- `Assets/AppEngine/Module/IAPModule/IAPUtil.cs`
- `Assets/Scripts/UI/ShopUI.cs`
- `Assets/Scripts/Managers/StarterPackManager.cs`

价值：

- Unity 与 Android 原生 SDK 双向通信。
- 支持 Banner、插屏、激励视频、开屏广告、去广告同步。
- 支持 Unity IAP 商品初始化、购买状态机、价格本地化、购买成功/失败事件分发。
- 支持礼包限时窗口、冷却、自动弹出、发奖。
- 支持 Firebase 埋点和 Remote Config A/B 配置。

### 6. 自动化构建基础

相关文件：

- `Assets/Editor/BuildScript.cs`
- `ProjectSettings/ProjectSettings.asset`
- `Assets/Plugins/Android`

价值：

- 支持 APK 构建。
- 支持 AAB 构建。
- 支持 Android Studio 工程导出。
- 配置 Android symbols，为崩溃定位提供基础。
- 已具备接入 Jenkins Unity batchmode 的入口。

## 技术亮点

### 启动流程状态机

为什么有价值：

移动游戏启动涉及资源、配置、SDK、首页和玩法入口，状态机能避免初始化顺序混乱。

面试官为什么关注：

面试官会重点看候选人是否能处理项目启动复杂度，而不是只会写单个 MonoBehaviour。

简历写法：

> 设计并维护 Unity 客户端启动流程状态机，拆分资源预加载、SDK 初始化、首页/玩法入口，提高启动流程可维护性和扩展性。

项目难点写法：

> 项目启动期需要同时处理词典 AssetBundle 加载、广告/IAP/Firebase 状态同步和首页跳转，需保证异步节点完成后再进入业务，避免黑屏、空数据和 SDK 未就绪问题。

### 业务与框架解耦

为什么有价值：

`IAppFlowBridge` 让框架层不直接依赖单词游戏业务，后续换首页、换玩法或新增流程节点更容易。

面试官为什么关注：

这是架构意识，不是单纯堆功能。

简历写法：

> 通过 Bridge 模式解耦框架启动流程与具体游戏业务，降低场景切换、资源预加载、首页入口之间的耦合。

项目难点写法：

> 启动流程既要服务框架，又要调用具体业务；若框架直接依赖玩法类，后续改动会影响全局启动链路。

### UGUI 多层 UI 框架

为什么有价值：

主界面、弹窗、引导、奖励、提示同时存在，多层 Canvas 能控制层级、输入和适配。

面试官为什么关注：

UGUI 框架、弹窗管理、事件清理是 Unity 高频面试题。

简历写法：

> 维护 UGUI 多层 UI 框架，支持主界面、弹窗、引导层分离，以及 UI 缓存、参数化打开、生命周期统一管理。

项目难点写法：

> 多弹窗和引导可能同时出现，需要保证层级正确、事件不穿透、关闭动画和销毁流程不冲突。

### UI 生命周期与事件清理

为什么有价值：

UI 关闭后继续收到事件是 Unity 常见线上问题，项目通过 `UIBase` 和 `GameEventMgr` 做了统一清理。

面试官为什么关注：

这体现你知道内存泄漏、重复事件、销毁后回调等真实项目问题。

简历写法：

> 统一 UI 生命周期和事件托管机制，降低弹窗重复打开、事件残留和销毁后回调导致的异常风险。

项目难点写法：

> 商店、礼包、引导、玩法 UI 都依赖全局事件，若没有统一清理机制，容易出现重复发奖、重复弹窗或 MissingReferenceException。

### 连线拼词热路径优化

为什么有价值：

连线输入每帧执行，直接影响 Android 真机流畅度。

面试官为什么关注：

会问 Update 中如何减少 GC、如何处理触摸、如何避免低端机卡顿。

简历写法：

> 实现移动端连线拼词交互，针对触摸检测、连线绘制和拼词字符串构建做低分配处理，保证 Android 真机流畅体验。

项目难点写法：

> 玩家拖动时每帧都要检测字母命中、更新连线和拼词显示，必须避免临时对象和字符串频繁分配。

### 道具状态一致性

为什么有价值：

Hint/Rocket 开格存在动画延迟，如果只看已打开格子，容易重复扣费或重复选中。

面试官为什么关注：

道具和经济系统是商业化游戏高风险区域。

简历写法：

> 实现 Hint/Target/Rocket 等关内道具逻辑，处理动画中 pending 状态、重复点击、结算边界和金币/道具消费一致性。

项目难点写法：

> 道具开格存在动画延迟，需记录已选但未提交的 pending 格子，避免快速连点造成重复消费或结算状态异常。

### AssetBundle 数据加载

为什么有价值：

词典和关卡数据量大，AssetBundle 比散 JSON 或硬编码更适合统一加载和更新。

面试官为什么关注：

资源管理、依赖、加载释放、Addressables 对比是中级 Unity 高频题。

简历写法：

> 搭建词典和关卡配置的 AssetBundle 加载链路，支持运行时预加载、多表数据解析和关卡数据源切换。

项目难点写法：

> 单词游戏依赖大量词库和关卡数据，直接放 Resources 或散文件会影响包体、加载和维护，需要统一打表和加载。

### Editor 内容生产工具

为什么有价值：

Editor 工具能提升团队效率，是 3~5 年 Unity 工程师的重要加分项。

面试官为什么关注：

能写工具说明你理解内容生产流程，而不是只写运行时代码。

简历写法：

> 开发 Unity Editor 关卡与配置工具，支持 CSV 打表、ScriptableObject 生成、AssetBundle 导出、关卡批量生成和外部 Wordscapes JSON 导入。

项目难点写法：

> 关卡量大、词典数据复杂，人工配置容易出错，需要工具化校验、排序、导入、批量生成和缺失词检测。

### Android 原生交互

为什么有价值：

项目中广告、Remote Config、Firebase、通知、Adjust 等都涉及 Unity 调 Android 原生。

面试官为什么关注：

商业 Unity 项目经常卡在 Android SDK 接入、Manifest、Gradle、回调和生命周期。

简历写法：

> 负责 Unity 与 Android 原生 SDK 桥接，封装广告、Firebase、Remote Config、Adjust、通知权限及 Activity 生命周期相关调用。

项目难点写法：

> Unity 和原生 SDK 生命周期不同，需要处理初始化顺序、空实例、回调转发、前后台切换和支付期间广告干扰。

### 广告变现策略

为什么有价值：

广告不是简单展示，还包含解锁等级、去广告、Banner 布局、激励奖励、开屏同步。

面试官为什么关注：

广告稳定性和展示策略直接影响收入与留存。

简历写法：

> 接入并封装 Android 广告 SDK，支持 Banner、插屏、激励视频、开屏广告、去广告状态同步和广告展示等级控制。

项目难点写法：

> 广告展示需兼顾收益和体验，需避免新手期过早打扰、购买流程被开屏广告打断、Banner 遮挡玩法 UI。

### Unity IAP 支付闭环

为什么有价值：

项目有商品配置、初始化状态、pending purchase、购买成功/失败、价格本地化、发奖逻辑。

面试官为什么关注：

IAP 是高风险模块，会重点问重复发奖、初始化失败、购买中断和非消耗品处理。

简历写法：

> 封装 Unity IAP 支付流程，维护初始化/购买状态机，支持商品价格本地化、购买事件分发、礼包/金币/去广告发奖。

项目难点写法：

> 支付链路涉及商店连接、商品拉取、用户中断、回调延迟、重复点击和广告状态同步，任何一步异常都可能影响收入。

### Firebase 与 Remote Config

为什么有价值：

埋点和远程配置支撑运营分析、A/B 测试和线上调参。

面试官为什么关注：

商业项目需要客户端理解事件设计、漏斗分析和实验边界。

简历写法：

> 接入 Firebase Analytics 与 Remote Config，支持关卡、商店、道具、教程等行为埋点，以及关卡数据源和字体配置 A/B 测试。

项目难点写法：

> 配置不能在关卡中途随意刷新，否则会导致数据源变化和关卡状态不一致，因此需要缓存和场景边界控制。

### 本地存档加密与兼容

为什么有价值：

项目不是散落写 PlayerPrefs，而是通过 `RecordDataManager` 集中管理，并兼容旧数据。

面试官为什么关注：

常见问题是版本更新后丢档、key 混乱、作弊篡改。

简历写法：

> 封装本地存档系统，支持金币、关卡、道具、礼包、引导等数据持久化，并加入轻量加密和旧版本数据兼容。

项目难点写法：

> 存档 key 多且关系复杂，升级加密方案时不能丢失老用户进度，需要读取新格式失败后兼容旧明文数据。

### 自动化构建基础

为什么有价值：

项目已有 Unity 构建菜单和方法，可被 Jenkins batchmode 调用。

面试官为什么关注：

3~5 年客户端通常要懂 CI、AAB、签名、符号表和构建产物归档。

简历写法：

> 维护 Unity Android 构建脚本，支持 APK/AAB 构建、Android Studio 工程导出和符号文件生成，为 Jenkins 自动化构建接入提供基础。

项目难点写法：

> Android 构建涉及场景列表、签名、AAB/APK、Gradle 导出、符号文件和 SDK 依赖，手工操作容易漏步骤。

### 包体与资源优化意识

为什么有价值：

项目使用 AAB、AssetBundle、Gradle 模板、Proguard 文件和 Unity 压缩相关配置，已经具备包体优化入口。

面试官为什么关注：

移动游戏首包大小影响下载转化，SDK 和资源膨胀是常见问题。

简历写法：

> 参与 Android 包体和资源构建链路优化，使用 AAB、AssetBundle 和 Unity/Gradle 配置管理资源与依赖。

项目难点写法：

> 词库、关卡、SDK、音频、特效资源多，包体容易膨胀，需要区分首包必需资源、表数据打包和 Android 依赖裁剪。

### 崩溃分析基础

为什么有价值：

`BuildScript` 已开启 Android symbols，说明构建链路考虑了符号文件，为 Native/IL2CPP 崩溃定位打基础。

面试官为什么关注：

线上崩溃定位是移动端必问，尤其是 C#、Java、Native 三类崩溃区别。

简历写法：

> 参与 Android 构建符号配置和异常日志排查流程，具备 Unity C#、Android Java、Native 崩溃定位基础。

项目难点写法：

> Unity Android 崩溃可能来自 C# 异常、Java SDK、Native so 或资源加载，需要结合 logcat、符号表和构建版本定位。

注意：当前仓库未看到完整 Crashlytics 崩溃闭环，面试中不要说“独立搭建线上 Crashlytics 体系”，可以说“具备基础，后续可补齐 Crashlytics 和符号上传”。

### ANR 分析意识

为什么有价值：

启动期 AssetBundle 加载、SDK 初始化、Android 原生桥接都是 ANR 高风险点。

面试官为什么关注：

Android 游戏常见 ANR 来自主线程 IO、同步等待 SDK、资源加载峰值。

简历写法：

> 具备 Android ANR 排查经验，关注 Unity 主线程 IO、SDK 初始化阻塞、资源加载峰值等问题。

项目难点写法：

> 启动期同时加载词典 AssetBundle 和初始化多个 SDK，如果同步阻塞主线程，会导致首启卡死或 ANR，需要异步化和分阶段加载。

注意：当前仓库未看到完整 ANR 监控闭环，建议作为面试扩展能力表达，不要包装成完整线上体系。

## 高频面试问题与回答方向

1. 项目启动流程是怎样的？
   - 回答方向：`GameEntrance` 配置 `App` 和 `ProcedureFSM`，依次执行 Launch、Preload、InitSDK、StartGame，业务通过 `WordGameFlowBridge` 接入。

2. 为什么要做流程状态机？
   - 回答方向：拆分异步阶段，便于控制加载顺序、进度条、SDK 初始化和异常兜底。

3. 拼词结束后的判定流程是什么？
   - 回答方向：`WordGuessUI` 收集输入并发送事件，`WordMainUI` 判断重复词、新词、额外词、错误词，`LevelData` 更新状态。

4. 如何恢复关卡进度？
   - 回答方向：`LevelStateVo` 保存 opened index 和 right spell list，通过 `RecordDataManager` 持久化。

5. UI 事件泄漏如何避免？
   - 回答方向：`UIBase` 内部事件管理器在 Hide/Close/Destroy 时 Clear，IAP 事件在 OnEnable/OnDisable 成对订阅解绑。

6. UGUI 安全区和平板适配如何处理？
   - 回答方向：根据 `Screen.safeArea` 设置 anchors，并根据宽高比调整 CanvasScaler match。

7. 连线玩法如何减少 GC？
   - 回答方向：使用 `StringBuilder`、列表缓存、对象池，避免 Update 中 LINQ、临时字符串和频繁 new。

8. 道具快速连点怎么处理？
   - 回答方向：使用 `_pendingOpenIndices` 记录动画中格子，避免重复开格和重复扣费。

9. 关卡数据从哪里来？
   - 回答方向：`DicManager` 从 AssetBundle 加载 `DicTable`、`WordsTable`、`WordsDictionaryTable`。

10. AssetBundle 和 Addressables 区别？
    - 回答方向：Addressables 是 AssetBundle 上层管理，提供地址、依赖、缓存、版本和远程更新能力。

11. PlayerPrefs 有什么风险？
    - 回答方向：易被篡改和清除，项目通过统一封装、轻量加密和旧数据兼容降低风险。

12. Android 原生 SDK 如何接入？
    - 回答方向：Unity 通过 `AndroidJavaClass` 调原生，原生通过 UnitySendMessage 或回调对象通知 `ReceiveJavaCall`。

13. 激励视频如何保证发奖安全？
    - 回答方向：只在成功回调后发奖，失败恢复 UI 状态，防止重复点击和重复发奖。

14. IAP 初始化中点击购买怎么办？
    - 回答方向：记录 pending product，初始化成功后继续购买，购买中禁用按钮。

15. 购买期间为什么要屏蔽开屏广告？
    - 回答方向：避免支付流程被开屏广告打断，通过 `SetIsProcessingIap` 同步原生状态。

16. Firebase 埋点怎么设计？
    - 回答方向：围绕关卡、教程、商店、购买、道具建立事件，带 `level_id`、`method`、`std_key` 等参数。

17. Remote Config 为什么不能关卡中途刷新？
    - 回答方向：避免关卡数据源变化导致当前状态与配置不一致。

18. Jenkins 如何接 Unity Android 构建？
    - 回答方向：Unity batchmode 调用 `BuildScript.BuildAab`，配置 Unity 路径、license、keystore、日志和产物归档。

19. Android 崩溃如何分析？
    - 回答方向：C# 看 Unity 异常，Java 看 logcat/Crashlytics，Native 看 tombstone 和 symbols。

20. ANR 如何分析？
    - 回答方向：看主线程堆栈、logcat、traces，重点排查启动同步 IO、SDK 初始化、资源加载。

## 简历可写内容

- 参与 Unity 移动端英文拼词休闲游戏客户端开发，负责核心连词玩法、关卡状态、道具系统、结算流程及 UI 动效实现。
- 搭建/维护启动流程 FSM、场景流转、UI 分层管理、事件分发、本地存档和资源加载等客户端基础框架。
- 接入 Unity IAP、Android 原生广告、Firebase Analytics、Remote Config、Google Review，实现商店、去广告、礼包、激励视频奖励和 A/B 配置能力。
- 设计关卡运行数据结构，支持普通关卡、每日挑战、额外词、词典解释、关卡进度恢复和旧存档兼容。
- 开发 Unity Editor 内容生产工具，支持 CSV 打表、ScriptableObject/AssetBundle 导出、关卡编辑、批量生成和外部关卡 JSON 导入。
- 针对 Android 移动端优化 UI 适配、广告展示策略、对象池复用、动画清理和高频输入路径，降低 GC 与异常状态风险。
- 维护 Unity Android 构建脚本，支持 APK/AAB 构建、Android Studio 工程导出和符号文件生成，为自动化构建接入提供基础。

## 项目难点总结

- 启动期需要同时处理资源加载、SDK 初始化、远程配置和首页跳转，异步顺序复杂。
- 拼词玩法中输入、动画、数据更新、结算判断存在时序问题。
- 道具开格有动画延迟，需防止重复选格、重复扣费和结算后继续消费。
- UGUI 弹窗多、层级复杂，需要统一生命周期、事件清理和适配。
- 词库和关卡数据量大，需要 Editor 工具和 AssetBundle 链路降低人工成本。
- Android 广告、IAP、Firebase、Remote Config 等 SDK 生命周期复杂，需处理原生回调和前后台状态。
- 商业化系统涉及收入，广告展示、IAP 发奖、去广告同步必须保证稳定。
- Android 构建涉及 AAB/APK、签名、Gradle、符号表和 SDK 依赖，适合接入 CI 自动化。

## 需要补充学习的知识点

### Unity 与 UGUI

- Canvas rebuild、Overdraw、DrawCall、图集、字体、Layout 性能。
- 弹窗栈管理、输入屏蔽、遮罩层、关闭动画和对象销毁顺序。
- DOTween 生命周期管理：`DOKill`、回调对象销毁、Sequence 清理。

### Android

- Unity Android 导出工程结构。
- Gradle 模板、Manifest 合并、Proguard/R8、依赖冲突。
- `AndroidJavaClass` / `AndroidJavaObject` 调用成本和异常处理。
- Activity 生命周期、前后台切换、权限申请。

### Jenkins

- Unity batchmode 构建命令。
- 参数化构建 APK/AAB。
- keystore 和密码安全管理。
- 构建日志、符号文件、AAB/APK 归档。
- 构建失败通知和版本号自动递增。

### 广告 SDK

- 激励视频、插屏、Banner、开屏广告展示策略。
- 填充率、展示率、eCPM、ARPDAU、LTV。
- 广告加载失败、回调丢失、重复点击和发奖兜底。

### Firebase

- Analytics 事件设计。
- Remote Config 缓存策略。
- A/B Testing 实验指标。
- Crashlytics 崩溃上报与符号上传。

### IAP

- 商品类型：Consumable、NonConsumable、Subscription。
- Pending purchase、恢复购买、重复发奖防护。
- Google Play receipt 校验。
- 支付期间广告和前后台状态处理。

### Addressables / AssetBundle

- AssetBundle 依赖、Manifest、缓存、版本。
- Addressables 分组、地址、远程 Catalog、缓存清理。
- 资源加载释放和引用计数。

### 性能优化

- Profiler、Memory Profiler、Frame Debugger。
- GC Alloc、对象池、热路径低分配。
- 纹理压缩、音频压缩、粒子数量、UI Overdraw。
- 首包大小、启动耗时、低端机适配。

### 崩溃与 ANR

- logcat、tombstone、traces.txt。
- C# 异常、Java 崩溃、Native 崩溃区别。
- Android symbols、mapping、Crashlytics 上传。
- 主线程阻塞、同步 IO、SDK 初始化耗时分析。

## 面试表达总纲

这个项目不要只说“我做了 UI 和功能”。更好的表达是：

> 我参与的是一个完整商业化 Unity 移动游戏客户端，工作覆盖核心玩法、UGUI 框架、资源数据链路、Editor 内容工具、Android 原生 SDK、广告变现、Unity IAP、Firebase 运营配置、Android 构建发布和线上问题排查基础。

这句话能把你的经历从“功能开发”提升到“完整移动游戏客户端工程经验”。
