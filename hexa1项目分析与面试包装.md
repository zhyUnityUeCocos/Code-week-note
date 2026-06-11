# Unity 3D 益智项目分析与面试包装

本文档基于当前 HexaSort Unity 项目的代码结构整理，目标是把项目中已经做过、但容易被低估的技术工作，转化为简历亮点、项目难点和面试表达。

## 一、项目简介

该项目是一个 Unity 3D 休闲益智项目，玩法接近 Hexa Sort / 六边形堆叠分类消除。

玩家将底部生成的方块堆拖拽到 3D 六边形底座上，相邻同色堆会合并，顶部连续同色达到条件后消除得分。同时项目支持木板、冰块、珍珠、火箭篮、小车、帘子、小球篮、草地等特殊关卡目标，并围绕这些目标设计了道具、复活、广告、IAP、活动和关卡难度系统。

主要代码依据：

- `Assets/Scripts/Layers/GameLayer.cs`
- `Assets/Scripts/Game/BlockHeapScript.cs`
- `Assets/Scripts/Game/BlockBaseManager.cs`
- `Assets/Scripts/Game/BlockBaseScript.cs`
- `Assets/Scripts/Game/LevelRuntimeBuilder.cs`
- `Assets/Scripts/entity/LevelEntity.cs`
- `Assets/Editor/BlockBaseEditor.cs`
- `Assets/Editor/LevelBatchExporter.cs`
- `Assets/Editor/LevelSimBatchRunner.cs`

## 二、技术栈

确定存在：

- Unity 3D + C# + UGUI。
- DOTween：拖拽回弹、合并飞行、UI 动画、路径动画、特效动画。
- Newtonsoft.Json + JsonUtility：关卡、配置、结算数据序列化。
- Resources 资源加载：关卡 JSON、配置表、Prefab、音效、图集。
- Unity IAP：商品购买、去广告、礼包、猪猪银行等。
- Firebase：Analytics、Remote Config、UserProperty、FCM 相关能力。
- Android 原生交互：平台工具、SDK 回调、设备信息、震动、广告桥接。
- EditorWindow：关卡编辑、批量导入导出、模拟器批跑、构建脚本。
- AssetBundle 构建脚本：关卡 Bundle 标记、Android Bundle 构建、version.json 生成。

需要谨慎表述：

- 当前未发现 `Assets/Addressables` 目录，也未发现 Addressables 运行时接入。
- 当前主流程仍以 `Resources.Load` 为主，AssetBundle 更像已有构建基础，是否线上启用需要进一步确认。
- 未发现 Jenkins 配置文件或流水线脚本，可以说“了解如何接入 Jenkins”，不要说“主导 Jenkins 自动化流水线”。

## 三、整体架构

项目整体不是严格 MVC/MVVM，而是典型的轻量手游客户端结构：

- 启动层：`LaunchScript` 负责启动初始化、低端机设置、体力计时、AB 配置、用户属性上报。
- UI 层：`LayerManager` + `BaseLayer<T>` 从 `Resources/Prefabs/Layers` 动态加载界面。
- 玩法层：`GameLayer` 管关卡生命周期、相机、棋盘旋转、出块、结算；`BlockHeapScript` 管拖拽；`BlockBaseManager` 管合并、消除、特殊块和胜负。
- 数据层：`entity/*Entity.cs` 从 `Resources/datas/*.json` 加载配置；`GameData` 和 `SecurePlayerPrefs` 管本地存档。
- 工具层：`Assets/Editor` 负责关卡编辑、JSON 导入导出、Excel 转 JSON、图集处理、Android 构建、关卡模拟。
- SDK 层：`Third` 和 `Services` 封装广告、IAP、Firebase、Remote Config、服务端时间、玩家数据。

这类架构的优点是开发快、接入成本低、适合中小型休闲项目。缺点是 Manager 和 Layer 容易变重，例如 `GameLayer` 和 `BlockBaseManager` 都承担了较多职责。

## 四、核心模块

### 1. 玩法核心

- `GameLayer`：关卡创建、旋转控制、底部出块、分数、通关失败结算、动态难度。
- `BlockHeapScript`：底部待放堆的拖拽、Raycast 检测、放置和回弹。
- `BlockBaseManager`：底座集合管理、同色合并、特殊块触发、死亡和通关检查。
- `BlockBaseScript`：单个底座状态、解锁、消除、文本显示、材质反馈、特殊块表现。
- `LevelRuntimeBuilder`：根据 JSON 构建运行时 3D 关卡。

### 2. 关卡数据

- `LevelData`：关卡号、地图号、难度、过关目标、出块权重、颜色权重、预设出块。
- `LevelDTO` / `CellDTO`：格子坐标、解锁类型、障碍参数、初始堆颜色。
- `Resources/LevelJson`：约 600+ 个关卡 JSON。
- `Resources/datas/level*.json`：多套难度表。

### 3. UI 与商业化

- `GameUILayer`：游戏内目标进度、道具按钮、目标展示、特殊块提示。
- `LevelSuccessLayer` / `LevelFailLayer` / `LevelFailLayerAbt`：通关、失败、复活、广告、重开。
- `PurchaseLayer`、`ShopLayer`、`RemoveAdLayer`、`StarterPackLayer`、`PiggyBankLayer`、`ZhuanPanLayer`、`WinStreakLayer`：商业化和活动模块。

### 4. 工程工具

- `BlockBaseEditor`：单关编辑、特殊块配置、JSON 导出。
- `LevelBatchExporter`：Prefab / JSON 双向批量转换。
- `LevelSimBatchRunner`：Editor 下批量跑关卡模拟，输出 CSV。
- `BuildScript`：Android Project / APK 构建。
- `BuildLevelBundles`：关卡 AssetBundle 构建和 version.json 生成。

## 五、3D 玩法系统

确定存在：

- 拖拽：`BlockHeapScript.OnMouseDown / OnMouseDrag / OnMouseUp`。
- 棋盘旋转：`GameLayer.Update` 监听拖动，并吸附到 0、60、120、180、240、300、360 度。
- Raycast：`RayUtil.getEndPoint` 从拖动物体位置向下发射射线检测 `BlockBase`。
- 选中反馈：`BlockBaseManager.setChoiceBlockBase` 调用 `BlockBaseScript.setIsChoiced` 修改材质参数。
- 落子逻辑：将堆里的子方块转移到目标底座的 `blocks` 节点。
- 同色合并：`BlockBaseManager.checkColorMerge` 找相邻顶部同色堆并触发合并。
- 消除计分：顶部连续同色达到条件后 `removeBlock` 消除并加分。
- 胜负判断：分数和所有特殊目标满足后通关；无可选空格后失败。

需要谨慎表述：

- 核心玩法主要是逻辑驱动，不是 Rigidbody 真实物理。
- 未发现正式撤销系统。
- 未发现正式最佳步提示系统，但有新手引导线和特殊块说明。
- 跳关更像调试/GM 能力，普通玩家路径需要进一步确认。

## 六、关卡系统

关卡系统是项目里非常值得包装的亮点。

### 关卡结构

`LevelData` 负责关卡目标和生成策略：

- `level`
- `map`
- `difficulty`
- `passScore`
- `passBoard`
- `passIce`
- `passPearl`
- `passGrass`
- `passCar`
- `passBall`
- `blockPercentage`
- `block`
- `colorPercentage`
- `color`
- `cacheStack`

`LevelDTO` 负责地图布局和初始状态：

- `levelId`
- `globalScale`
- `cells`
- `posX`
- `posZ`
- `unlockType`
- `unlockScore`
- `unlockBoard`
- `unlockRocket`
- `hasGrass`
- `carDirection`
- `ballColors`
- `childBlocks`

### 加载流程

1. `GameLayer.createLevel()` 获取当前关卡号。
2. `LevelEntity.GetLevelDifficultyDataByTableName` 根据当前难度表取 `LevelData`。
3. `LevelRuntimeBuilder.TryLoadJsonByMap(levelData.map)` 加载 `Resources/LevelJson/level{map}` 或 `LevelJson2`。
4. `LevelRuntimeBuilder.BuildFromJson` 生成运行时底座、障碍和初始堆。
5. `BlockBaseManager` 初始化底座列表和邻接逻辑。
6. `GameLayer.createBlockHeap` 按权重或 `cacheStack` 生成底部待放堆。

### 难度系统

项目支持多套难度表：

- `level`
- `level_1`
- `level_2`
- `level_3`
- `level_sort`

并且存在动态难度策略：

- 前 20 关固定 easy。
- 后续根据玩家步数、当前进度、baseline 进度差调整出块权重。
- 支持 AB 配置动态难度策略和阈值。

## 七、UI 与商业化系统

项目的商业化系统比较完整，不只是“有广告按钮”。

确定存在：

- 激励广告：复活、广告格解锁、金币、部分道具/奖励。
- 插屏广告：通关、重开等节点。
- IAP：金币包、去广告、StarterPack、FestivalPack、LevelFailPack、PiggyBank。
- 活动：连胜、排行榜、转盘、猪猪银行、节日礼包、评分弹窗。
- AB 测试：失败页变体、复活模式、猪猪银行、关卡通道、动态难度策略。
- Firebase 埋点：通关、失败、购买、广告、关卡进入、活动等。

可以包装的价值：

- 你做的不只是玩法，还参与了休闲手游的“玩法 + 广告 + IAP + 活动 + 数据”闭环。
- 面试官会非常关注这个，因为商业项目最终要看留存、变现、活动运营和线上调参能力。

## 八、工程与工具系统

### 编辑器扩展

已做内容：

- 单关可视化编辑。
- 特殊块类型配置。
- 方块堆配置。
- JSON 导出。
- Prefab / JSON 双向批量转换。
- 关卡模拟工具窗口。

价值：

- 降低策划和开发重复劳动。
- 让关卡数据可批量生产、批量验证、批量回归。
- 体现工具链意识，这是 3~5 年 Unity 工程师的重要能力。

### 自动化构建

已做内容：

- `BuildScript` 支持 Android Studio Project 导出。
- `BuildScript` 支持 APK 构建。
- 设置 Android 符号表为 Public。
- `BuildLevelBundles` 支持关卡 AssetBundle 构建。

谨慎表述：

- 当前未看到 Jenkins 配置。
- 可以说项目已有自动化构建入口，Jenkins 可通过 Unity batchmode 调用这些静态方法。

## 九、性能优化点

确定存在：

- 低端机关闭阴影。
- 低端机关闭抗锯齿。
- 部分消除动画在低端机降级。
- 设置 `Application.targetFrameRate = 60`。
- 设置 `QualitySettings.vSyncCount = 0`。
- 使用 SpriteAtlas。
- 使用 `ObjectPool` 缓存 Prefab 和部分对象。
- 切场景时清理对象池。
- 部分流程调用 `Resources.UnloadUnusedAssets`。
- 文本透明度使用 `MaterialPropertyBlock`，避免直接修改共享材质。

需要进一步确认：

- 真实 DrawCall、Batches、Mesh 数、材质数、内存占用、加载耗时需要用 Profiler / Frame Debugger 实测。
- GPU Instancing 未看到明确使用。
- Addressables 未看到接入。

## 十、你做了但容易低估的技术点

### 1. 数据驱动关卡

为什么有价值：

- 关卡数量很多时，手动维护 Prefab 成本高。
- 数据驱动可以支持批量生产、批量调整、AB 切换和线上配置。

面试官为什么关注：

- 休闲益智项目最核心的产能就是关卡。
- 面试官会判断你是否理解“内容生产效率”。

简历亮点：

- 搭建配置表 + JSON 驱动的 3D 关卡加载流程，支持 600+ 关卡运行时生成。

项目难点：

- 解决关卡数量增长后 Prefab 维护成本高、版本切换困难、特殊块兼容复杂的问题。

### 2. 运行时关卡构建

为什么有价值：

- 将 JSON 数据转成真实 3D 场景对象。
- 支持缺格、障碍、初始堆、特殊表现统一生成。

面试官为什么关注：

- 这是玩法底层能力，不是简单 UI 开发。

简历亮点：

- 实现关卡 JSON 到 3D 场景对象的运行时构建，支持多种特殊格与初始堆状态还原。

项目难点：

- 特殊块表现、逻辑初始化、旧数据兼容、资源缺失兜底都要处理。

### 3. 关卡编辑器工具

为什么有价值：

- 提高策划生产效率。
- 降低开发反复改 Prefab 的成本。

面试官为什么关注：

- 工具链能力是 3~5 年 Unity 工程师的加分项。

简历亮点：

- 开发 Unity Editor 关卡编辑/导入导出工具，支持关卡 Prefab 与 JSON 双向转换。

项目难点：

- 从手动摆关升级为可编辑、可导出、可批处理的关卡生产流程。

### 4. 关卡模拟和胜率分析

为什么有价值：

- 关卡难度不能只靠人工试玩。
- 批量模拟能快速发现过难、过易或异常关卡。

面试官为什么关注：

- 体现你能用工程手段解决策划调优问题。

简历亮点：

- 实现关卡纯逻辑模拟与蒙特卡洛批量测试，输出胜率和平均步数数据。

项目难点：

- 把真实玩法从 MonoBehaviour 和表现层中抽离成可复现的纯逻辑模拟。

### 5. 动态难度策略

为什么有价值：

- 休闲游戏需要控制挫败感和挑战感。
- 可根据玩家表现动态调整后续出块。

面试官为什么关注：

- 难度曲线直接影响留存。

简历亮点：

- 参与动态难度系统，基于玩家进度差调整关卡出块权重。

项目难点：

- 避免难度频繁抖动，同时让玩家感知自然。

### 6. 3D 拖拽 + Raycast 交互

为什么有价值：

- 3D 交互比普通 UGUI 点击复杂。
- 棋盘旋转后仍要保证落点准确。

面试官为什么关注：

- Unity 玩法岗位必问输入、射线、坐标转换、碰撞层。

简历亮点：

- 实现 3D 拖拽落子、Raycast 选格、落点高亮和回弹反馈。

项目难点：

- 玩家操作是屏幕坐标，落点是旋转后的世界空间 3D 棋盘。

### 7. 特殊块系统

为什么有价值：

- 特殊块扩展了关卡目标和玩法变化。
- 能支撑后续持续加关卡机制。

面试官为什么关注：

- 会看你如何管理状态、扩展类型、同步表现和逻辑。

简历亮点：

- 维护多类型特殊格逻辑，支持解锁、击打、动画、目标进度与通关判定。

项目难点：

- 特殊块既影响表现，也影响合并、消除、目标和胜负判断。

### 8. 道具系统

为什么有价值：

- 道具是休闲游戏变现的重要入口。
- 道具要直接介入 3D 棋盘状态。

面试官为什么关注：

- 会追问道具和核心玩法如何解耦、如何避免状态冲突。

简历亮点：

- 实现锤子、交换、刷新等道具，并接入金币、广告消耗和埋点。

项目难点：

- 道具使用期间要禁用普通拖拽/旋转，并和合并动画、UI、Collider 状态互斥。

### 9. 广告变现链路

为什么有价值：

- 激励广告和插屏是休闲游戏主要收入来源。
- 广告不是单点 SDK，而是和玩法、复活、奖励、失败页深度结合。

面试官为什么关注：

- 商业项目面试很关注广告闭环和回调安全。

简历亮点：

- 接入激励广告/插屏广告场景，完成奖励发放、失败兜底、购买入口和埋点。

项目难点：

- 广告回调可能失败、延迟、重复，奖励发放需要防重复和兜底。

### 10. Firebase + Remote Config

为什么有价值：

- 支持线上埋点、用户属性、AB 实验和远程调参。

面试官为什么关注：

- 说明你接触过线上运营体系，而不只是本地 Demo。

简历亮点：

- 接入 Firebase Analytics / UserProperty / Remote Config，支持 AB 配置下发和关键行为埋点。

项目难点：

- 配置生效时机要谨慎，不能在关卡中途刷新数据导致当前局状态异常。

### 11. IAP 支付系统

为什么有价值：

- 支付模块风险高，涉及发货、掉单、恢复购买和数据一致性。

面试官为什么关注：

- 会重点看你是否理解支付幂等和校验流程。

简历亮点：

- 接入 Unity IAP，处理商品购买、去广告、礼包、恢复购买和购买成功回调。

项目难点：

- 防止重复发货、漏发货、掉单，以及客户端和服务端状态不一致。

### 12. Android 原生交互

为什么有价值：

- 移动端 Unity 项目经常需要接 Android SDK、设备信息、广告回调。

面试官为什么关注：

- Unity 客户端岗位常问 AndroidJavaClass、UnitySendMessage、生命周期。

简历亮点：

- 封装 Android 原生能力调用，包括广告、震动、设备信息和 SDK 回调。

项目难点：

- Java / Unity 回调时序、生命周期、异常兜底和多 SDK 冲突。

### 13. 自动化构建基础

为什么有价值：

- 项目交付需要稳定可复现的构建流程。

面试官为什么关注：

- 3~5 年工程师需要有交付链路意识。

简历亮点：

- 编写 Unity Editor Android 构建脚本，支持 Gradle 工程导出和 APK 生成。

项目难点：

- 构建参数、签名、符号表、多环境配置和构建产物管理。

### 14. AssetBundle 构建基础

为什么有价值：

- 关卡资源可以按包管理，为分包、热更新、远程下载打基础。

面试官为什么关注：

- 会考察资源依赖、版本 hash、缓存更新。

简历亮点：

- 搭建关卡 AssetBundle 构建流程，生成 Bundle hash 清单用于版本管理。

项目难点：

- 资源依赖、版本对齐、缓存清理、失败回滚。

### 15. 资源管理

为什么有价值：

- 休闲游戏体量小但关卡和特效多，资源常驻会影响内存和包体。

面试官为什么关注：

- 会问 Resources、AssetBundle、Addressables 的区别和取舍。

简历亮点：

- 维护 Resources 加载、Prefab 缓存、对象池和资源释放流程。

项目难点：

- 避免频繁 Instantiate / Destroy、避免资源常驻、避免材质实例膨胀。

### 16. 低端机性能适配

为什么有价值：

- Android 休闲游戏用户设备差异大。

面试官为什么关注：

- 移动端项目非常看重真机性能经验。

简历亮点：

- 针对低端 Android 机型做阴影、抗锯齿、动画和特效降级。

项目难点：

- 既要降低性能开销，又不能明显牺牲表现和手感。

### 17. UGUI Layer 管理

为什么有价值：

- 商业化休闲游戏弹窗多，界面生命周期复杂。

面试官为什么关注：

- 会问 UI 打开关闭、事件穿透、弹窗优先级、资源释放。

简历亮点：

- 搭建 Layer 式 UI 管理，支持界面动态加载、关闭、参数传递和交互锁。

项目难点：

- 多弹窗互斥、关闭残留、按钮连点、事件穿透。

### 18. 本地存档安全

为什么有价值：

- 金币、体力、道具、关卡进度都是核心经济数据。

面试官为什么关注：

- 休闲游戏会关心防作弊、数据迁移和版本兼容。

简历亮点：

- 对金币、体力、关卡、道具等核心存档做加密迁移和持久化。

项目难点：

- 老版本数据迁移失败可能导致玩家资产丢失。

### 19. 日志和异常兜底

为什么有价值：

- 线上问题定位依赖日志、异常捕获和符号表。

面试官为什么关注：

- 会问崩溃怎么查、ANR 怎么定位、线上日志怎么保留。

简历亮点：

- 封装日志工具，并在关键流程增加异常捕获和错误追踪。

项目难点：

- Release 关闭普通日志后，仍要保留关键错误信息和可定位上下文。

### 20. 崩溃 / ANR 分析基础

为什么有价值：

- Android 线上项目必然会遇到崩溃、卡死、SDK 异常。

面试官为什么关注：

- 这是移动端工程经验的重要分水岭。

简历亮点：

- 谨慎写法：具备 Android 崩溃日志、符号表、Logcat 和 SDK 回调问题排查经验。

项目难点：

- Unity C#、Java、Native、第三方 SDK 混合调用时，堆栈和复现路径可能不完整。

## 十一、适合简历的写法

可以写：

- 参与 Unity 3D 休闲益智项目核心玩法开发，实现拖拽落子、Raycast 选格、棋盘旋转、同色合并和多目标胜负判定。
- 搭建数据驱动关卡系统，支持配置表 + JSON 生成 3D 棋盘，覆盖 600+ 关卡资源。
- 开发 Unity Editor 关卡编辑器与批量导入导出工具，支持关卡 Prefab / JSON 双向转换，提升关卡制作效率。
- 实现关卡纯逻辑模拟与批量胜率分析工具，用于辅助关卡难度调优。
- 维护锤子、交换、刷新、复活等道具系统，并接入广告、金币消耗、埋点和 UI 状态控制。
- 接入 Firebase Analytics / Remote Config，支持 AB 测试、用户属性上报和远程参数控制。
- 参与广告 SDK、Unity IAP、Android 原生桥接等商业化模块开发，完成激励广告、插屏、去广告和礼包购买流程。
- 针对 Android 低端机做性能适配，包括阴影/抗锯齿降级、对象池、图集、资源释放和动画降级策略。

谨慎写：

- Jenkins：当前项目没有明确 Jenkins 配置，可以说“了解 Jenkins + Unity batchmode 自动化构建流程”，不要说主导完整流水线。
- Addressables：当前项目未发现 Addressables 接入，可以说“了解 Addressables，并能对当前 Resources / AssetBundle 方案做迁移设计”。
- 崩溃平台：项目有 Cloud Diagnostics 包、符号表配置和日志工具，但不要夸大成完整崩溃平台主导经验。

## 十二、STAR 案例

### 案例 1：关卡数据化改造

S：关卡数量不断增加，Prefab 维护成本高，特殊块新增后旧关卡兼容复杂。  
T：让关卡可以由数据驱动生成，并保留编辑器生产流程。  
A：设计 `LevelDTO / CellDTO` 数据结构，运行时用 `LevelRuntimeBuilder` 生成 3D 棋盘；编辑器支持 Prefab / JSON 批量转换。  
R：关卡可批量生成、批量调整、批量回归，方便 AB 关卡通道和难度表复用。

### 案例 2：3D 拖拽落子体验优化

S：玩家在旋转后的 3D 棋盘上拖拽方块，落点容易不准。  
T：保证拖拽手感、落点准确和选中反馈稳定。  
A：使用世界空间拖拽位置 + 向下 Raycast 检测底座，结合材质高亮和回弹动画。  
R：拖拽交互稳定，适配 3D 棋盘旋转和不同视角。

### 案例 3：多特殊块合并消除联动

S：关卡中存在木板、冰、珍珠、火箭、小车、小球等特殊目标。  
T：让普通合并、特殊块解锁、目标进度和胜负判定统一协作。  
A：在 `BlockBaseManager` 中统一调度合并链和特殊块触发，在 `GameUILayer` 同步目标进度。  
R：支持多目标关卡，玩法扩展能力增强。

### 案例 4：关卡难度模拟工具

S：大量关卡依靠人工试玩成本高，难以及时发现难度异常。  
T：用数据辅助策划评估胜率和平均步数。  
A：抽离纯逻辑模拟器，提供 EditorWindow 批量运行并输出 CSV。  
R：提高关卡调优效率，让难度调整更有依据。

### 案例 5：商业化闭环接入

S：道具、复活、广告、IAP、活动弹窗都影响玩家局内外流程。  
T：将广告奖励、金币购买、复活清盘、失败重开和埋点串成闭环。  
A：在失败页、游戏 UI、道具系统、广告回调、GameData 中完成状态同步。  
R：支持激励广告、插屏、礼包、去广告等核心变现路径。

## 十三、30 个高频面试问题

| # | 问题 | 追问方向 | 优秀回答方向 |
|---|---|---|---|
| 1 | 这个项目整体架构是什么？ | 为什么不用纯 MVC/MVVM？ | Layer UI + Singleton Manager + Entity 配置表 + Game Runtime。 |
| 2 | `GameLayer` 负责什么？ | 它是不是职责过重？ | 管关卡生命周期、旋转、出块、结算；可继续拆分控制器。 |
| 3 | 3D 拖拽落子怎么实现？ | 为什么用向下 Raycast？ | 世界空间拖拽 + 向下射线，适合旋转棋盘后的落点检测。 |
| 4 | 如何判断格子可放置？ | 特殊块能不能放？ | 空堆且 `Free` 才能放。 |
| 5 | 合并逻辑怎么做？ | 两堆和三堆怎么选目标？ | 查找相邻顶部同色堆，按纯色、杂色、特殊块策略合并。 |
| 6 | 通关条件是什么？ | 多目标如何避免漏判？ | 分数和所有特殊目标都满足后通关。 |
| 7 | 失败条件是什么？ | 棋盘满但可合并怎么办？ | 合并链完成后再判断是否无空格。 |
| 8 | 项目用了 Unity 物理吗？ | Rigidbody 有必要吗？ | 核心是 Collider + Raycast，逻辑驱动，不依赖 Rigidbody。 |
| 9 | 棋盘旋转怎么处理？ | 为什么吸附 60 度？ | 六边形方向，拖动旋转后吸附最近角度。 |
| 10 | 道具系统怎么设计？ | 锤子/交换/刷新区别？ | 锤子打击，交换移动堆，刷新重建底部堆。 |
| 11 | UGUI Layer 怎么管理？ | 关闭 UI 怎么防残留？ | `LayerManager` 动态加载，`BaseLayer<T>` 单例管理。 |
| 12 | 如何避免点 UI 时误触 3D？ | 手机上怎么处理？ | `EventSystem.IsPointerOverGameObject`，并处理 touch fingerId。 |
| 13 | 图集有什么作用？ | 怎么减少 DrawCall？ | SpriteAtlas 让 UI 使用同图集材质，利于合批。 |
| 14 | UI 怎么适配？ | 刘海屏怎么做？ | CanvasScaler、锚点、安全区、宽屏节点。 |
| 15 | 多语言怎么实现？ | 切语言如何刷新？ | `LanguageManager` + `language.json` + `ILanguage`。 |
| 16 | Android 构建流程是什么？ | 符号表怎么处理？ | Editor 构建脚本 + Gradle + APK + Public symbols。 |
| 17 | Jenkins 怎么接？ | Unity 命令行怎么写？ | batchmode 调用静态构建方法，归档 APK/符号表/日志。 |
| 18 | Keystore 怎么管理？ | 密码能不能写代码？ | 不应硬编码，应用 Jenkins Credentials 或环境变量。 |
| 19 | Android 原生交互怎么做？ | 回调丢失怎么办？ | AndroidJavaClass / UnitySendMessage，回调要幂等。 |
| 20 | 低端机优化做了什么？ | 如何分档？ | RAM 分档关闭阴影、抗锯齿、动画降级。 |
| 21 | 对象池怎么用？ | 当前不足是什么？ | 缓存 Prefab 和对象，但玩法块仍可继续池化。 |
| 22 | `Resources.Load` 有什么问题？ | 怎么迁移？ | 资源全进包、依赖不透明、卸载难；可迁 Addressables。 |
| 23 | AssetBundle 当前怎么做？ | version.json 有什么用？ | 构建关卡 Bundle，并用 hash 管版本。 |
| 24 | Addressables 和 AssetBundle 区别？ | 热更新选哪个？ | Addressables 是更高层资源系统，底层可用 AssetBundle。 |
| 25 | 广告 SDK 怎么接？ | 如何防重发奖励？ | 奖励基于回调结果，状态锁和幂等保护。 |
| 26 | 插屏广告放在哪？ | 如何不影响体验？ | 通关/重开等节点，结合新手保护、冷却、去广告。 |
| 27 | Firebase 用在哪里？ | 事件怎么设计？ | Analytics、UserProperty、RemoteConfig，事件命名稳定。 |
| 28 | Remote Config 怎么生效？ | 游戏中切表安全吗？ | 关卡外刷新配置，避免局内破坏状态。 |
| 29 | IAP 流程是什么？ | 掉单怎么办？ | 初始化、购买、校验、发货、Confirm、Restore、幂等。 |
| 30 | 崩溃 / ANR 怎么分析？ | Unity 崩溃怎么定位？ | Logcat、符号表、堆栈、主线程阻塞、SDK 回调分析。 |

## 十四、需要补充学习的知识点

优先级高：

- Unity Profiler：CPU、GC、Rendering、Memory、Timeline。
- Frame Debugger：DrawCall、Batches、材质切换、UI 合批。
- Memory Profiler：贴图、Mesh、材质实例、常驻资源、内存快照对比。
- Addressables：Catalog、Remote Load Path、缓存、依赖、资源释放。
- AssetBundle：依赖收集、Manifest、Hash、缓存、版本回滚。
- Jenkins + Unity batchmode：自动构建、参数化、多渠道、符号表归档。
- Android Crash / ANR：Logcat、Java Crash、Native Crash、主线程阻塞、符号还原。
- 广告 SDK：激励广告回调幂等、无填充、加载/展示时机、频控。
- IAP：Receipt 校验、掉单、重复发货、Restore、服务端校验。
- UGUI 性能：Canvas 拆分、Raycaster、Mask、ScrollRect、SpriteAtlas。
- 材质性能：`material`、`sharedMaterial`、`MaterialPropertyBlock` 的区别。

## 十五、面试表达建议

推荐表达方式：

> 这个项目我不是只做单个界面或单个功能，而是参与了从 3D 核心玩法、关卡数据化、编辑器工具、道具、广告/IAP、Firebase 埋点到 Android 构建和低端机优化的一整套休闲手游客户端流程。

关于 Jenkins / Addressables / 崩溃平台，推荐这样说：

> 当前项目里已有 Android 构建脚本和 AssetBundle 构建基础，Jenkins 没有在仓库中看到完整流水线配置。如果要接入，我会用 Unity batchmode 调用静态构建方法，并归档 APK/AAB、符号表和构建日志。Addressables 当前没有正式接入，主流程仍是 Resources，所以我更熟悉 Resources / AssetBundle 的现状和迁移方案。

这样既真实，又能体现你知道下一步怎么工程化。
