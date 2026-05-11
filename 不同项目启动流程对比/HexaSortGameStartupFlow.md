# HexaSort 启动流程文档

> 生成时间：2026-02-26  
> 唯一场景：`Assets/Scenes/MainScene.unity`  
> 界面切换方式：`LayerManager.showLayer()` 动态加载/销毁 Prefab，无场景切换

---

## 总览流程图

```
App 启动
  │
  ├─ [Awake]       设备配置 · 安全存储迁移 · SDK JNI 桥接
  │
  ├─ [Start]       日常数据检查 · AB 配置拉取（异步）· 玩家信息上传
  │
  ├─ [GameLoadLayer]  进度条软等待（1~2s）→ GC 清理
  │
  ├─ [Firebase 回调]  Remote Config 就绪，AB 值锁定
  │
  └─ [MainLayer / GameLayer]
          └─ CheckMainLayerPopup() 弹窗优先级责任链
```

---

## 一、Awake 阶段

> 场景所有 MonoBehaviour 的 `Awake` 同时触发，无严格顺序保证。

### LaunchScript.Awake()

| 调用 | 作用 |
|------|------|
| `GameData.getRequestId()` | 生成/读取设备唯一请求 ID |
| `Input.multiTouchEnabled = false` | 禁用多点触摸 |
| `Screen.sleepTimeout = NeverSleep` | 禁止设备息屏 |
| `CultureInfo("en-US")` | 主线程强制英文数字格式，避免浮点解析异常 |
| `GameData.addOpenCount()` | 启动次数累计 +1 |
| `GameData.setFirstOpenTime()` | 记录首次启动时间戳 |
| `MigrateToSecureStorage()` | 旧版 PlayerPrefs → 加密存储迁移（**一次性，含版本标记**） |

**迁移字段**（`MigrateToSecureStorage`，key: `SecurePrefs_Migrated_v1`）：

```
gold · heart · CurPlayLevel · CurBuildIsland
PropChuiZiCount · PropSwapCount · PropRefreshCount
ItemUseCount · PassLevelCount · UseZhuanPanCount · hasAdRemove · BuyNoAdDate
```

### 其他 Awake

| 类 | 作用 |
|----|------|
| `RemoteConfigUtil.Awake()` | 建立 Firebase Remote Config JNI 桥接（`RemoteConfigLibrary`） |
| `AdAndroidUtil.Awake()` | 广告 SDK 初始化 |
| `BaseLayer<T>.Awake()` | 各界面 Layer 单例注册，调用 `Initialize()` |

---

## 二、Start 阶段

### LaunchScript.Start()

```
GameData.UpdateEventTimer()           // 更新活动计时器
GameData.CheckDailyReset()            // 检查每日重置

InvokeRepeating("onInvokeSecond", 0.5f, 1f)  // 启动心脏恢复 1 秒心跳

QualitySettings.antiAliasing = 0     // 关闭抗锯齿（低内存设备判断）
Application.targetFrameRate = 60     // 目标帧率 60
DOTween.SetTweensCapacity(1250, 50)  // 预分配 Tween 容量

AdAndroidUtil.SetNewUserShowReAdFlag()  // 首次启动不主动拉激励广告

ABTestingMgr.Instance.Refresh()      // 拉取 Firebase Remote Config AB 值（异步）
SavePlayerOnFirstOpenAsync()         // 上传玩家信息 + 申请 FCM 推送 Token
SetUserProperty()                    // 首次初始化 Firebase 用户属性（一次性）
```

### 心跳 onInvokeSecond（每秒）

- 心脏未满时计算恢复时间，达到 `Consts.resumeHeartTime` 后执行 `GameData.changeHeart()`
- 刷新 `MainLayer / LevelFailLayer / GetHeartLayer / RetryConfirmLayer` 中的心脏倒计时 UI

### Update（每帧）

- 每秒调用 `GameData.UpdateEventTimer()`
- 每 60 分钟调用 `GameData.CheckDailyReset()`
- `ServerTimeManager.Update(Time.deltaTime)`

---

## 三、加载过渡层（GameLoadLayer）

场景中预先放置，作为启动画面（进度条 + 岛屿图）。

```
Initialize()
  └── island.sprite = 当前建造岛屿图片

Start()（协程）
  ├── ABTestingMgr.IsReady == false → doLoadAni(duration: 2f)  // 等待 Remote Config
  └── ABTestingMgr.IsReady == true  → doLoadAni(duration: 1f)

loadEnd()
  ├── GC.Collect() + Resources.UnloadUnusedAssets()   // 主动清理内存
  ├── isToGame == true  → LayerManager.showLayer(GameLayer)   // 重开关卡
  └── isToGame == false → LayerManager.showLayer(MainLayer)   // 正常进主界面
                           └── 0.2s 后 CheckMainLayerPopup()
```

> `isToGame` 由外部（关卡失败重试等流程）赋值为 `true`，默认 `false`。

---

## 四、Firebase Remote Config 回调（异步，时机不定）

```
Java 端拉取完成
  └── ReceiveJavaCall.OnFetchFirebaseRemoteConfig()
        └── ABTestingMgr.Instance.Ready()
              ├── isReady = true
              └── 下次 Refresh() 时锁定 AB 值（GetReadyValue = true，防止重复拉取）
```

**AB 测试字段**：

| Key | 用途 |
|-----|------|
| `level_data` | 关卡数据表选择（level.json / level_1.json 等） |
| `ui_main_background` | 主界面背景风格 |
| `hexa_style` | 六角块样式 |
| `island_build` | 岛屿建造功能开关 |
| `adrevenue_uploadbar` | 广告收入上报阈值 |

---

## 五、主界面初始化（MainLayer.Initialize）

```
读取头像 Sprite
CheckAndShowHasAdRemoveToast()      // 去广告购买状态检测

岛屿判断：
  IsIslandUnlockAll() → 切换下一岛屿

关卡表边界检查：
  curLevel > LevelList.last → 隐藏开始按钮，显示 commingSoon

IslandLayer 未存在 → showLayer(IslandLayer)

UI 刷新：
  refreshUI()    // 金币 / 心脏 / 关卡数
  refreshProgress()  // 排行榜 / 连胜进度条
```

---

## 六、弹窗优先级责任链（CheckMainLayerPopup）

> 任意一个条件命中后立即 `return`，保证同时只弹一个弹窗。

| 优先级 | 条件 | 弹出 |
|--------|------|------|
| 1 | 有本地未结算过关结果 | `LevelSuccessLayer` |
| 2 | 有岛屿积分待解锁 && 关卡 > 2 | `AddPartScoreLayer` |
| 3 | 首次关卡 > 3 且未评分 | `RatingLayer` |
| 4 | 排行榜超时未领奖 | `RankMainLayer` |
| 5 | 圣诞节/万圣节排行榜活动首次 | `RankLayer`（节日） |
| 6 | 连胜事件活动 | `WinStreakLayer` |
| 7 | StarterPack 礼包未领取 | `StarterPackLayer` |
| 8 | 节日礼包活动 | `FestivalLayer` |
| — | 无命中 | 正常显示主界面 |

---

## 七、关键设计说明

### 单场景架构
项目只有 `MainScene` 一个 Unity 场景，所有"页面"均通过 `LayerManager.showLayer()` 动态实例化/销毁 Prefab 实现。

### AB 测试软等待
`GameLoadLayer` 进度条时间区分 1s（已就绪）和 2s（未就绪），用动画时间换取 Remote Config 拉取窗口，不做强阻塞，避免卡死启动流程。

### 数据安全迁移
`MigrateToSecureStorage()` 有幂等保护（`SecurePrefs_Migrated_v1` 标记），多次启动只执行一次，确保不重复迁移覆盖数据。

### 用户属性初始化
`SetUserProperty()` 同样有一次性标记（`UserProperty_Set_v1`），仅在第一次启动时向 Firebase 批量上报用户属性，后续由各业务模块按需更新。

---

## 相关文件索引

| 文件 | 职责 |
|------|------|
| `Assets/Scripts/common/LaunchScript.cs` | 启动总控，设备配置，心跳 |
| `Assets/Scripts/Layers/GameLoadLayer.cs` | 加载过渡动画，进入主界面或游戏 |
| `Assets/Scripts/Layers/Base/BaseLayer.cs` | 所有 Layer 的基类，单例管理 |
| `Assets/Scripts/Layers/MainLayer.cs` | 主界面初始化，弹窗优先级入口 |
| `Assets/Scripts/Utils/LayerManager.cs` | 界面路由，`CheckMainLayerPopup` 弹窗链 |
| `Assets/Scripts/Third/ABTestingMgr.cs` | AB 测试值管理，Remote Config 缓存 |
| `Assets/Scripts/Third/RemoteConfigUtil.cs` | Firebase Remote Config JNI 桥接 |
| `Assets/Scripts/Utils/ReceiveJavaCall.cs` | Java→Unity 回调接收（广告、Firebase） |
| `Assets/Scripts/common/GameData.cs` | 全局玩家数据读写（PlayerPrefs 封装） |
