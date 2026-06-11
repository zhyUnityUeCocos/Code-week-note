# Unity 3D Hexa Sort 项目分析、简历包装与面试准备文档

> 说明：本文档根据当前仓库代码结构、资源目录、关键脚本和 Git 提交历史整理。  
> 其中“确定存在”表示能在代码或提交中找到明确依据；“推测/需要确认”表示根据结构和命名判断，但仍建议你结合真实工作经历校准。  
> Git 职责推断部分默认你的提交作者为 `cb`。如果你的 Git author 不是 `cb`，需要重新按对应 author 分析。

---

## 1. 项目简介

这是一个 Unity 3D 移动端休闲益智项目，玩法接近 **Hexa Sort / 六边形颜色堆叠消除**。

玩家从底部拖拽一组方块堆到 3D 六边形棋盘的空底座上。落子后，系统根据相邻底座顶部颜色进行自动合并，连续同色数量达到阈值后触发消除、计分和特殊目标推进。项目同时包含广告、IAP、Firebase、RemoteConfig、排行榜、连胜、礼包等商业化和运营模块。

确定存在的项目特征：

- 3D 六边形棋盘玩法。
- 玩家拖拽、旋转棋盘、点击特殊底座、使用道具。
- 基于 `Resources/LevelJson` 的大量关卡 JSON。
- `Assets/Editor` 下有关卡编辑、导入导出、批量模拟工具。
- Android 广告、IAP、Firebase、RemoteConfig、Adjust 等商业化接入。
- 低端机适配、Tween 生命周期、资源释放、Timer 回调防护等稳定性优化。

---

## 2. 技术栈

确定存在：

- Unity 3D
- C#
- UGUI
- DOTween
- Newtonsoft.Json
- Resources 资源加载
- SpriteAtlas 图集
- AndroidJavaClass / AndroidJavaObject 原生桥接
- Firebase Analytics / RemoteConfig
- AdMob 或公司广告 SDK 封装
- Unity IAP
- PlayerPrefs 本地存档
- AssetBundle 构建工具雏形
- Unity Memory Profiler
- Android Logcat

项目资源规模概览：

- C# 脚本约 166 个。
- `Assets/Resources/LevelJson` 约 611 个关卡 JSON。
- 材质约 102 个。
- Shader 约 41 个。
- 模型约 40 个。
- 图片约 693 张。

---

## 3. 目录结构与整体架构

重点目录：

- `Assets/Scripts/Game`：玩法核心。
- `Assets/Scripts/Layers`：UI 层、游戏界面、结算、失败、商店、活动、排行榜等。
- `Assets/Scripts/Managers`：UI、存档、玩法预演等管理器。
- `Assets/Scripts/Entity`：配置表实体，如关卡、活动、排行榜、礼包、语言等。
- `Assets/Scripts/Third`：第三方 SDK 接入，如广告、Firebase、IAP、RemoteConfig。
- `Assets/Scripts/Utils`：通用工具、模拟器、对象池、平台桥接、日志、语言、计时器等。
- `Assets/Editor`：关卡工具、构建工具、Excel 转 JSON、图集工具、批量模拟工具。
- `Assets/Resources/datas`：配置表 JSON。
- `Assets/Resources/LevelJson`：棋盘结构 JSON。
- `Assets/Resources/Prefabs`：UI、游戏对象、特效、关卡标准 Prefab。
- `Assets/StreamingAssets`：视频和 Google 配置文件。

整体分层：

1. 启动与全局层  
   `LaunchScript`、`GlobalConfig`、`Consts`、`Singleton`、`SingletonMono`

2. 场景与 UI 层  
   `LevelManager`、`UIManager`、`LayerManager`、`BaseLayer`

3. 玩法核心层  
   `GameLayer`、`BlockHeapScript`、`BlockBaseScript`、`BlockBaseManager`、`BlockBaseTopColliderScript`

4. 数据配置层  
   `LevelEntity`、`LevelBaseLineEntity`、`ConfigEntity`、`GameDataManager`

5. 商业化与运营层  
   `AdAndroidUtil`、`FirebaseUtil`、`IAPUtil`、`RemoteConfigUtil`、`ABTestingMgr`

6. 工具链层  
   `LevelBatchImporter`、`LevelBatchExporter`、`BlockBaseEditor`、`LevelSimBatchRunner`、`ExcelToJson`

---

## 4. 核心模块分析

### 4.1 GameLayer

位置：`Assets/Scripts/Layers/GameLayer.cs`

职责：

- 创建和重置关卡。
- 读取关卡配置。
- 调用 `LevelRuntimeBuilder` 从 JSON 创建棋盘。
- 生成底部待放置方块堆。
- 处理棋盘旋转。
- 控制玩家输入状态。
- 处理动态难度。
- 关卡成功/失败结算。
- 埋点上报。

关键逻辑：

- `CreateLevel()`：创建当前关卡。
- `CreateBlockHeap()`：生成底部待放置块堆。
- `OnPlayerMoveCompleted()`：玩家完成一次落子后计步，并在特定步数触发预演和动态难度判断。
- `CompleteTargetScore()`：胜利处理。
- `ShowGameResult()`：显示胜利/失败结果。

### 4.2 BlockHeapScript

位置：`Assets/Scripts/Game/BlockHeapScript.cs`

职责：

- 控制玩家拖拽底部方块堆。
- 检测 UI 点击穿透。
- 通过 Raycast 判断落点。
- 将方块堆移动到目标底座。
- 通知 `BlockBaseManager` 开始合并检测。
- 成功落子后触发新方块堆生成。

确定存在的操作：

- `OnMouseDown`
- `OnMouseDrag`
- `OnMouseUp`
- `EventSystem.current.IsPointerOverGameObject`
- `BlockBaseManager.CheckBlockBaseByRay`

### 4.3 BlockBaseScript

位置：`Assets/Scripts/Game/BlockBaseScript.cs`

职责：

- 单个棋盘底座的状态管理。
- 底座解锁类型维护。
- 方块堆数量显示。
- 消除逻辑。
- 特殊块逻辑：Score、Ad、Board、Ice、Zhenzhu。
- 选中高亮。
- 消除特效、飞行特效、珍珠特效。

重要能力：

- `IsCanChoice()`：判断底座是否可落子。
- `IsCanRemove()`：判断是否满足消除。
- `RemoveBlock()`：执行消除动画和分数增加。
- `Unlock()`：解锁底座或特殊目标。
- `HitBoard()`：击打木板。
- `Jiedong()`：冰块解冻。
- `FlyZhenzhu()`：珍珠收集飞行动画。

### 4.4 BlockBaseManager

位置：`Assets/Scripts/Game/BlockBaseManager.cs`

职责：

- 管理所有底座。
- 统一进行 Raycast 命中结果处理。
- 判断邻接同色底座。
- 调度合并动画。
- 检查消除。
- 检查胜利和失败。
- 控制合并期间输入状态。

关键逻辑：

- `CheckColorMerge()`：扫描可合并底座。
- `JudgeMergeLogic()`：选择合并方向。
- `MergeBlock()`：执行方块翻转飞行动画。
- `CheckRemoveBlock()`：检查是否满 10 消除。
- `CheckPassLevel()`：胜利判断。
- `CheckIsDie()`：失败判断。

### 4.5 GamePreviewManager

位置：`Assets/Scripts/Managers/GamePreviewManager.cs`

职责：

- 通过棋盘快照模拟玩家落子后的结果。
- 不修改真实场景对象。
- 预估分数、合并次数、消除数量、木板/冰块/珍珠剩余数量。
- 给动态难度系统提供判断依据。

技术点：

- 使用 `GameStateSnapshot` 快照。
- 使用 `PreviewResult` 返回预演结果。
- 对 snapshot/result 做对象池，减少频繁分配。
- 用于每隔一定步数的动态难度评估。

### 4.6 LevelRuntimeBuilder

位置：`Assets/Scripts/Game/LevelRuntimeBuilder.cs`

职责：

- 定义 `LevelDTO`、`CellDTO`。
- 从 `Resources/LevelJson/level{map}` 加载关卡 JSON。
- 根据 JSON 运行时实例化标准底座。
- 回填解锁类型、初始堆叠、特殊块表现。
- 根据格子位置估算邻接距离。

---

## 5. 3D 玩法系统

### 5.1 玩家操作方式

确定存在：

- 拖拽底部块堆到棋盘空格。
- 横向滑动旋转棋盘。
- 使用锤子道具点击目标。
- 使用交换道具拖动底座堆到另一个底座。
- 点击广告底座观看广告解锁。
- 点击重开、回主界面、复活、下一关等 UI。

### 5.2 Raycast 检测

核心链路：

1. `BlockHeapScript.OnMouseDrag/OnMouseUp`
2. `BlockBaseManager.CheckBlockBaseByRay`
3. `RayUtil.GetEndPoint`
4. `Physics.Raycast`
5. 命中 `BlockBase` tag 的 Collider

Raycast 主要用于判断当前拖拽块堆落在哪个底座上。

### 5.3 碰撞与物理系统

确定存在：

- `MeshCollider`
- `Physics.Raycast`
- `OnMouseDown/OnMouseDrag/OnMouseUp`

未发现核心玩法依赖：

- `Rigidbody`
- `OnCollision`
- `OnTrigger`

结论：项目核心玩法不是刚体物理模拟，而是基于 Collider 命中和 Transform 节点转移的逻辑棋盘玩法。

### 5.4 合并与消除逻辑

流程：

1. 玩家放置块堆。
2. 检查相邻底座顶部颜色是否一致。
3. 根据底座纯色/杂色、相邻关系、特殊块数量选择合并方向。
4. 将源底座顶部连续同色块移动到目标底座。
5. 播放翻转、飞行、音效。
6. 检查目标底座是否顶部连续同色达到 10。
7. 达到则消除，增加分数。
8. 消除影响周围特殊块：木板、冰块、珍珠。
9. 再次递归检查合并。
10. 若无合并/消除，检查胜负。

### 5.5 胜负判断

胜利条件：

- 当前分数大于等于关卡目标分数。
- 木板目标剩余为 0。
- 冰块目标剩余为 0。
- 珍珠目标剩余为 0。

失败条件：

- 所有 Free 底座都不为空，没有可放置空间。

### 5.6 道具系统

确定存在三类主要道具：

1. 锤子  
   可以清除 Free 底座上的块，也可以击打木板、冰块、珍珠目标。

2. 交换  
   将两个 Free 底座上的堆叠互换。

3. 刷新  
   清空底部待放置块堆并重新生成。

道具入口在 `GameUILayer`，具体目标交互在 `BlockBaseTopColliderScript`。

### 5.7 提示、重开、跳关、撤销

确定存在：

- 新手引导线。
- 特殊块首次出现引导。
- 道具解锁提示。
- 重开逻辑。
- GM 直接胜利/失败。

需要进一步确认：

- 正式撤销功能未发现。
- 正式跳关功能未发现。

---

## 6. 关卡系统

### 6.1 两层关卡数据

第一层：关卡目标与出块配置  
位置：`Assets/Resources/datas/level.json`、`level_1.json`、`level_2.json` 等。

关键字段：

- `level`
- `map`
- `passScore`
- `passBoard`
- `passIce`
- `passPearl`
- `blockPercentage`
- `block`
- `colorPercentage`
- `color`
- `cacheStack`

第二层：棋盘结构  
位置：`Assets/Resources/LevelJson/levelN.json`

关键字段：

- `levelId`
- `globalScale`
- `cells`
- `posX`
- `posZ`
- `unlockType`
- `unlockScore`
- `unlockBoard`
- `childBlocks`

### 6.2 关卡加载流程

1. `GameLayer.CreateLevel`
2. 根据当前难度选择关卡表。
3. `LevelEntity.GetLevelDifficultyDataByTableName`
4. 读取 `levelData.map`
5. `LevelRuntimeBuilder.TryLoadJsonByLevel`
6. `Resources.Load<TextAsset>("LevelJson/level{map}")`
7. `LevelRuntimeBuilder.BuildFromJson`
8. 实例化标准底座。
9. 生成方块堆和特殊块表现。
10. `BlockBaseManager.CalibrateAndInit`

### 6.3 关卡编辑器工具

确定存在：

- `BlockBaseEditor`：手动编辑底座、颜色块、解锁类型。
- `LevelBatchExporter`：批量导出关卡 JSON。
- `LevelBatchImporter`：从 JSON 初始化当前选中的棋盘。
- `LevelSimBatchRunner`：关卡批量模拟。
- `ExcelToJson`：表格转 JSON。
- `SimCsvReplayer`：CSV 回放。

### 6.4 是否数据驱动

确定是数据驱动。

棋盘结构、目标条件、出块权重、颜色数量、预设出块、难度表、baseline 均由 JSON 或配置表驱动。

---

## 7. UI 与商业化系统

确定存在的 UI：

- 主界面
- 游戏界面
- 成功结算
- 失败复活
- 商店
- 购买
- 去广告
- 恢复购买
- 设置
- 语言
- 排行榜
- 连胜活动
- 转盘
- 礼包
- GM
- 特殊块引导
- 道具解锁

商业化模块：

- 激励视频
- 插屏/全屏广告
- Banner
- 去广告
- 金币购买
- 复活
- 三倍金币
- StarterPack
- LevelFailPack
- BasicPack
- FestivalPack
- WinStreak 奖励
- Rank 奖励

Firebase 埋点覆盖：

- 关卡进入
- 关卡成功/失败
- moves 步数
- 复活入口
- 复活点击
- 道具使用
- 道具购买
- 广告解锁
- 购买成功
- 活动入口
- 用户属性

---

## 8. Android 与 SDK 接入

确定存在：

- `AndroidJavaClass`
- `AndroidJavaObject`
- Unity 调 Java 静态方法。
- Java 回调 Unity 的接收类 `ReceiveJavaCall`。
- Android Manifest。
- Gradle 模板。
- Google Play Review。
- Firebase 配置。
- IAP。
- Adjust。
- RemoteConfig。
- 广告 SDK 生命周期回调。

关键脚本：

- `AdAndroidUtil`
- `FirebaseUtil`
- `RemoteConfigUtil`
- `IAPUtil`
- `Platform_Android`
- `ReceiveJavaCall`

Android 相关提交中能看到：

- 公司 SDK 升级。
- Java 防崩溃措施。
- Unity singleTask 修改。
- Adjust 上报。
- RemoteConfig 调整。
- GM 使用 Android Java 调用判断。

---

## 9. 性能优化与稳定性

确定存在的优化：

- 低端机按内存阈值关闭灯光阴影。
- `Application.targetFrameRate = 60`。
- `QualitySettings.vSyncCount = 0`。
- `DOTween.SetTweensCapacity(1250, 50)`。
- 部分 Tween 使用 `SetRecyclable`。
- 部分 Tween 使用 `SetLink` 避免对象销毁后 Tween 悬挂。
- 关卡结束后主动释放资源。
- `Resources.UnloadUnusedAssets`。
- 对象池缓存 Prefab 和实例。
- SpriteAtlas 合图。
- Android ASTC_4x4 图集配置。
- `TimerUtil` 回调防 MissingReference 和 NullReference。
- `GamePreviewManager` 使用快照池和结果池。

需要进一步确认或补充实测：

- DrawCall 数量。
- Batches 数量。
- SetPass Calls。
- Mesh 数量。
- 材质实例数量。
- GPU Instancing 是否实际启用。
- 贴图压缩是否全部符合 Android 目标机。
- 加载耗时与内存峰值。
- ANR/Crash 线上分析流程。

---

## 10. 根据提交历史推断你的职责

默认你的 Git 作者为 `cb`。

### 10.1 你可能负责过的模块

强证据：

- 关卡与动态难度。
- 关卡工具链。
- 商业化 SDK 与埋点。
- Android SDK 和出包稳定性。
- UI 商业化流程。
- 性能与稳定性修复。

中等证据：

- Rank UI。
- WinStreak UI。
- 主界面活动入口。
- 礼包开放条件。
- 多地区配置。
- 结算页金币。
- 复活流程。

不建议过度包装：

- Jenkins 自动化。
- 完整 Addressables 远程热更。
- 深度 GPU Instancing 优化。
- 自研物理系统。

### 10.2 最适合写进简历的模块

优先级最高：

1. 关卡数据驱动与动态难度系统。
2. 关卡编辑器和批量模拟工具链。
3. 商业化 SDK 与 Firebase 埋点体系。
4. 移动端稳定性与性能优化。
5. 失败复活、道具、结算等商业化玩法闭环。

### 10.3 最能体现中级客户端能力的工作

- 能把玩法逻辑、关卡数据、UI、商业化、埋点串成完整闭环。
- 能做 Editor 工具，而不是只做界面。
- 能用纯逻辑模拟器和回放工具定位问题。
- 能处理 Android 原生 SDK 和 Unity 交互。
- 能关注性能、资源释放、低端机兼容、线上稳定性。

### 10.4 最能体现工程能力的工作

- JSON 关卡结构设计。
- 批量导入导出工具。
- Monte Carlo 批量模拟。
- CSV 回放。
- Android SDK 接入。
- RemoteConfig 和多地区配置。
- PlayerPrefs 安全封装。
- 资源释放和对象池。

### 10.5 最能体现问题排查能力的工作

- 动态难度异常排查。
- 关卡不可过/过难排查。
- 结算和复活埋点不准排查。
- Tween 悬挂和对象销毁回调异常排查。
- Android 原生 SDK 崩溃排查。
- 场景切换后资源残留排查。

---

## 11. STAR 项目案例

### 案例 1：关卡数据驱动与动态难度系统

**Situation**  
项目关卡数量多，玩法需要支持多难度表、多地图配置，并根据玩家进度动态调整出块策略。

**Task**  
将关卡结构、目标条件、出块权重和难度表解耦，支持按 `map` 加载关卡 JSON，同时实现前期关卡难度控制和中后期动态难度调整。

**Action**  
参与改造 `GameLayer`、`LevelEntity`、`LevelRuntimeBuilder`，实现 `levelData.map` 到 `Resources/LevelJson/level{map}` 的加载链路；接入 `level_easy/level_normal/level_hard` 等表，前 10 关固定 easy；通过玩家步数、预演结果和 baseline 进度差值判断是否切换难度。

**Result**  
关卡结构与难度配置解耦，支持同一关卡逻辑映射不同棋盘地图；出块策略从固定配置升级为基于玩家进度的动态调整。

简历描述：

- 参与 3D Hexa Sort 关卡数据驱动改造，基于 JSON 动态生成棋盘结构，并接入 easy/normal/hard 多难度表与 baseline 进度评估，实现按玩家表现动态调整出块策略。

### 案例 2：关卡编辑器与批量验证工具链

**Situation**  
项目存在数百个关卡，手动维护 Prefab 和人工验证关卡成本高，问题关卡定位效率低。

**Task**  
提供关卡 JSON 导入、批量模拟、单表测试和回放能力，提升关卡制作、验证和问题定位效率。

**Action**  
开发/维护 `LevelBatchImporter`、`LevelSimBatchRunner`、`LevelPureLogicSimulator`、`SimCsvReplayer`；支持 JSON 初始化棋盘、批量 Monte Carlo 模拟、CSV 回放和 GM 创建复盘控制器。

**Result**  
关卡可以批量导入、批量验证和复盘定位，减少人工调关和复现问题的成本。

简历描述：

- 搭建关卡编辑与验证工具链，支持关卡 JSON 导入/导出、批量模拟、CSV 回放和 GM 调试，辅助策划快速定位异常关卡和难度波动问题。

### 案例 3：纯逻辑预演与动态难度评估

**Situation**  
玩家落子后可能触发多轮合并、消除和特殊块目标变化，仅根据当前分数无法准确判断玩家实际进度。

**Task**  
在不影响真实棋盘的情况下，预估玩家落子后的合并链和目标完成度，为动态难度提供数据依据。

**Action**  
接入 `GamePreviewManager`，使用棋盘快照模拟多轮合并、消除、木板、冰块、珍珠变化；统计得分、剩余目标、合并次数等结果，并结合 `LevelBaseLineUtil` 计算玩家进度与基准进度差值。

**Result**  
出块策略具备数据依据，难度调整不再只依赖关卡号或固定概率。

简历描述：

- 实现基于棋盘快照的落子预演逻辑，模拟多轮合并与特殊目标变化，并结合关卡 baseline 计算玩家进度偏差，用于动态难度调节。

### 案例 4：商业化 SDK 与 Firebase 埋点体系

**Situation**  
项目需要上线运营，涉及广告、IAP、Firebase、Adjust、RemoteConfig、活动入口等商业化链路。

**Task**  
保证广告展示、购买、复活、结算、活动入口等关键行为可追踪、可配置、可结算。

**Action**  
维护 `FirebaseUtil`、`AdAndroidUtil`、`IAPUtil`，增加用户属性上报、胜负 moves 参数、关卡完成事件、Adjust token、RemoteConfig 和 IAP 回调处理。

**Result**  
形成从广告/IAP 到关卡结果、复活、道具使用、活动行为的埋点闭环，支撑运营分析和商业化调优。

简历描述：

- 负责移动端商业化接入与埋点维护，接入广告、IAP、Firebase 用户属性、Adjust 事件与 RemoteConfig，覆盖关卡开始/结束、复活、道具、购买等核心运营事件。

### 案例 5：移动端稳定性与性能问题排查

**Situation**  
移动端休闲项目存在低端机性能、Tween 残留、场景切换资源释放、对象销毁后回调异常等风险。

**Task**  
降低线上崩溃和卡顿风险，提高长时间游玩稳定性。

**Action**  
优化消除动画 Tween 生命周期，处理 Tween 回收；在场景结束后主动释放资源；为 Timer 回调增加 MissingReference/NullReference 防护；参与 Android Java 防崩溃和 Activity 配置调整。

**Result**  
降低对象销毁后回调、Tween 悬挂、场景切换残留资源、Android 原生层异常带来的稳定性风险。

简历描述：

- 针对移动端稳定性问题进行专项优化，处理 Tween 生命周期、场景资源释放、Timer 回调异常和 Android 原生层防崩溃问题，提升长时间运行稳定性。

---

## 12. 简历可直接使用描述

推荐版本：

- 参与 Unity 3D 休闲益智项目客户端开发，负责核心玩法、关卡系统、商业化 SDK、运营埋点及工具链维护。
- 参与关卡数据驱动改造，基于 JSON 动态生成六边形棋盘结构，支持按 map 加载关卡、多难度表配置和前期关卡难度锁定。
- 实现/维护基于棋盘快照的玩法预演系统，模拟落子后的合并链、消除结果和特殊目标变化，并结合 baseline 评估玩家进度用于动态难度调整。
- 开发 Unity Editor 关卡工具，支持关卡 JSON 导入、批量模拟、CSV 回放和 GM 调试，提升关卡制作、验证和问题定位效率。
- 接入并维护广告、IAP、Firebase、RemoteConfig、Adjust 等商业化模块，覆盖关卡、复活、道具、购买、活动等核心埋点链路。
- 针对移动端进行稳定性优化，处理 Tween 生命周期、Timer 回调异常、场景资源释放和 Android 原生层防崩溃问题。

偏 1-3 年版本：

- 参与 Unity 休闲益智手游客户端开发，负责关卡加载、道具逻辑、结算界面、广告/IAP 接入和运营埋点维护。
- 根据策划配置实现关卡 JSON 加载和棋盘生成，支持多难度表和特殊目标配置。
- 开发关卡导入、模拟和回放工具，辅助定位关卡难度和合并逻辑问题。
- 维护 Android 广告、IAP、Firebase 埋点和 RemoteConfig 接入，处理复活、购买、结算等商业化流程。
- 优化移动端运行稳定性，处理 Tween 残留、资源释放和对象销毁回调异常。

---

## 13. 面试高频问题、追问与优秀回答方向

### 13.1 Unity / 玩法

1. 这个项目的核心玩法是什么？  
   追问：玩家一次操作后的完整流程是什么？  
   优秀回答：拖拽块堆、Raycast 命中底座、放置、邻接同色合并、满 10 消除、更新目标和胜负。

2. 拖拽落点怎么检测？  
   追问：如何避免点到 UI 也触发 3D？  
   优秀回答：`OnMouseDrag/OnMouseUp` 配合 `Physics.Raycast`，同时用 `EventSystem.IsPointerOverGameObject` 屏蔽 UI。

3. 为什么核心玩法不用 Rigidbody？  
   追问：Collider 在这里起什么作用？  
   优秀回答：这是棋盘逻辑，不需要真实物理；Collider 用于 Raycast 和鼠标/触控事件命中。

4. 六边形格子的邻接怎么判断？  
   追问：距离判断有什么风险？  
   优秀回答：项目用 `Vector3.Distance <= blockDistance * 1.2f`；风险是缩放和位置误差，所以运行时根据格子位置估算邻接距离。

5. 合并链如何避免逻辑混乱？  
   追问：动画没播完玩家继续拖怎么办？  
   优秀回答：用 `isMergeEnd`、`isCanRotate`、`isCanDragBlockHeap` 控制输入锁，动画完成后回调递归检查。

6. 胜利和失败如何判断？  
   追问：特殊目标如何计入？  
   优秀回答：胜利要求分数、木板、冰块、珍珠目标全部完成；失败是没有 Free 且空的底座。

7. 道具系统怎么设计？  
   追问：锤子、交换、刷新分别怎么实现？  
   优秀回答：锤子清除或打特殊块；交换移动两个底座子节点；刷新重建底部块堆并扣道具。

8. 动态难度怎么做？  
   追问：baseline 起什么作用？  
   优秀回答：根据玩家步数和预演结果计算进度差，与 baseline 比较，决定出块难度表切换。

9. `GamePreviewManager` 有什么价值？  
   追问：为什么要做纯逻辑快照？  
   优秀回答：不改真实棋盘，模拟落子后的合并、消除、目标变化，用于难度评估和问题定位。

10. DOTween 在项目中怎么使用？  
    追问：对象销毁时 Tween 怎么处理？  
    优秀回答：用于合并飞行、缩放、UI 动画；需要 `Kill`、`SetLink`、`SetRecyclable` 避免悬挂和 GC。

### 13.2 UGUI

11. UGUI 层级怎么管理？  
    追问：如何避免界面互相覆盖？  
    优秀回答：通过 `UIManager/LayerManager` 统一加载 Layer，区分普通层、高层、弹窗层。

12. UI 点击和 3D 点击冲突怎么处理？  
    追问：移动端多指呢？  
    优秀回答：EventSystem 判断 UI 命中；项目启动关闭多点触控。

13. UGUI 性能优化做过什么？  
    追问：Canvas 拆分怎么考虑？  
    优秀回答：图集、减少 Layout 重建、控制 GraphicRaycaster、动静分离 Canvas、避免频繁 SetActive。

14. ScrollView 优化怎么做？  
    追问：排行榜列表如何优化？  
    优秀回答：复用 cell、减少动态 Layout、缓存图片和文本、只刷新可见项。

15. UI 合批失败常见原因？  
    追问：Mask 会有什么影响？  
    优秀回答：不同材质、贴图、Canvas、Mask、Shader、动态层级会打断合批；Mask 会增加额外渲染和裁剪开销。

### 13.3 Android

16. Android 原生 SDK 怎么接入 Unity？  
    追问：Java 回调 Unity 怎么做？  
    优秀回答：C# 用 `AndroidJavaClass/AndroidJavaObject` 调 Java；Java 可用 UnitySendMessage 或回调桥通知 C#。

17. Activity 生命周期对 Unity 有什么影响？  
    追问：广告 SDK 在 pause/resume 时要注意什么？  
    优秀回答：要处理前后台切换、广告暂停恢复、音频、计时器、SDK 生命周期，否则容易黑屏或回调异常。

18. Android Manifest 和 Gradle 模板改过什么？  
    追问：权限和广告 ID 怎么处理？  
    优秀回答：Manifest 配置权限、Activity、广告 ID；Gradle 模板接 SDK 依赖、Resolver、签名和 ABI。

19. ANR 怎么排查？  
    追问：Unity 侧和 Android 侧分别看什么？  
    优秀回答：看 Logcat、traces、主线程耗时、SDK 回调、资源加载；Unity 侧看 Profiler 和日志。

20. Android 原生崩溃怎么定位？  
    追问：符号表有什么用？  
    优秀回答：收集堆栈、Logcat、Crash 平台、符号表还原 native 堆栈，结合版本号和设备信息定位。

### 13.4 广告 SDK / Firebase / IAP

21. 激励视频奖励怎么保证正确发放？  
    追问：重复回调怎么办？  
    优秀回答：只在成功回调发奖，发奖要幂等，记录状态，防重复点击和重复回调。

22. 插屏和激励视频有什么区别？  
    追问：什么时候展示更合适？  
    优秀回答：激励视频必须用户主动触发并给奖励；插屏适合结算或切场景，但要控制频率。

23. Firebase 埋点怎么设计？  
    追问：如何保证事件可分析？  
    优秀回答：事件名稳定、参数统一、核心漏斗完整，如 `level_id`、`moves`、`method`、`std_key`。

24. Firebase UserProperty 有什么用？  
    追问：项目适合上报哪些属性？  
    优秀回答：用于分群分析；可上报关卡、金币、难度、连胜/连败、道具使用、复活、购买次数。

25. RemoteConfig 可以用来做什么？  
    追问：风险是什么？  
    优秀回答：调广告、活动、难度、AB 参数；风险是默认值、拉取失败、类型转换、灰度失控。

26. IAP 流程怎么保证安全？  
    追问：断网/重复发货怎么办？  
    优秀回答：初始化、拉商品、购买、回调、校验、发货、确认订单；发货要幂等并本地记录。

### 13.5 Addressables / AssetBundle / 性能优化

27. Resources 有什么问题？  
    追问：如果迁移 Addressables 怎么做？  
    优秀回答：Resources 不利于依赖分析、增量更新和卸载；可拆关卡、UI、特效 group，按场景加载并 Release。

28. AssetBundle 和 Addressables 区别？  
    追问：当前项目是什么状态？  
    优秀回答：AB 是底层打包方案；Addressables 负责地址、依赖、缓存、Catalog。当前项目主流程是 Resources，AB 有构建脚本雏形。

29. DrawCall/Batches 怎么优化？  
    追问：合批失败常见原因？  
    优秀回答：图集、共享材质、静态/动态合批、GPU Instancing；不同材质、Shader、Canvas、透明排序会打断合批。

30. GC Alloc 怎么排查？  
    追问：项目里哪些地方容易产生 GC？  
    优秀回答：Profiler 看 GC Alloc；注意 Update 中 new、字符串拼接、LINQ、闭包、频繁 Instantiate/Destroy、Tween 回调。

---

## 14. 需要补充学习的知识点

### Unity

- Profiler：CPU、Rendering、Memory、GC Alloc、Timeline。
- Frame Debugger：DrawCall、Batches、SetPass Calls、材质切换。
- DOTween 生命周期：`Kill`、`SetLink`、`SetRecyclable`。
- Collider、Raycast、Input、EventSystem 的关系。
- 配置表驱动、JSON、ScriptableObject 的取舍。

### UGUI

- Canvas 拆分原则。
- LayoutGroup、ContentSizeFitter 性能代价。
- GraphicRaycaster 优化。
- Mask、ScrollView、Overdraw 优化。
- 图集、字体、材质导致的合批问题。
- 弹窗层级与防点击穿透。

### Android

- AndroidJavaClass / AndroidJavaObject。
- Activity 生命周期。
- Manifest、Gradle 模板。
- Logcat、ANR traces、Crash 堆栈。
- Android 权限、广告 ID、网络状态。
- SDK 初始化顺序和回调线程。

### 广告 SDK

- 激励视频奖励幂等。
- 广告加载失败兜底。
- 插屏频控。
- 去广告状态同步。
- 广告收入归因参数。

### Firebase

- Event、Parameter、UserProperty 区别。
- 漏斗设计。
- RemoteConfig 默认值和拉取失败兜底。
- Crashlytics 或 Cloud Diagnostics 基础使用。

### IAP

- 商品初始化。
- 购买、恢复购买、订单确认。
- 发货幂等。
- 断网和重复回调处理。
- Google Play 订单校验基础。

### Addressables / AssetBundle

- Group、Label、Catalog、Remote Load Path。
- 依赖分析。
- 缓存与版本更新。
- `LoadAssetAsync` 和 `Release` 生命周期。
- 首包资源和远程资源拆分。
- Resources 迁移方案。

### 性能优化

- DrawCall、SetPass Calls、Batches 区别。
- Mesh、材质、贴图压缩、Shader Variant。
- 对象池适用边界。
- 低端机分级策略。
- 加载耗时、内存峰值、卡顿定位流程。

---

## 15. 面试时的推荐表达策略

### 可以大胆讲

- 关卡数据驱动。
- 运行时 JSON 构建棋盘。
- 动态难度。
- 关卡模拟和回放工具。
- Firebase、广告、IAP、RemoteConfig 接入。
- 移动端稳定性优化。

### 谨慎讲

- Addressables：当前项目证据不足，可以说“项目主流程仍用 Resources，了解迁移方案”。
- Jenkins：没有明确脚本，不建议说自己负责。
- GPU Instancing：没有明确核心使用，不建议说做过完整优化。
- Crash/ANR：可以说做过 Android 原生防崩溃和 Logcat 排查，不要说完整搭建线上崩溃平台。

### 面试中的一句总括

我在这个项目中不只是做 UI 或单点功能，而是参与了玩法、关卡数据、工具链和商业化闭环。比较有代表性的工作是关卡 JSON 运行时构建、动态难度预演、关卡批量模拟回放，以及 Android 广告/IAP/Firebase 接入和稳定性优化。

