# 游戏启动流程

## 启动流程图

```
【Loading.unity 场景启动】
        │
        ▼
【AddGameCore.Start()】
  ├─ 创建 GameObject "GameCore"   → AddComponent<GameCore>()   DontDestroyOnLoad
  └─ 创建 GameObject "GameManager" → AddComponent<GameManager>() DontDestroyOnLoad
        │
        ▼
【GameCore.Start()】
  判断当前场景 == Loading？
  是 → 直接 InitLoading()
  否 → SendNotice 切到 Loading 场景 → 再触发 InitLoading()
        │
        ▼
【InitLoading()】  ← GameCoreBase 模板方法，GameCore 重写
  ├─ Config()         → 确认需要加载字典 / 数据
  ├─ InitApp()        → 设置帧率、创建 MonoUtl / TimerUtl
  ├─ InitGUI()        → 设置字体、UI 缩放适配（FairyGUI）
  ├─ GuiPreloading()  → 加载 Sound / ComPreload 两个 UI 包
  └─ StartCoroutine(CoroutineInitLoading())
        │
        ▼
【CoroutineInitLoading()】  协程，有明确的时序节点
  ├─ [节点 1] SendNotice → 打开 LAYER_UI 模块（创建 UI 层级容器）
  │            yield return null
  ├─ [节点 2] SendNotice → 打开 LOADING_BAR_MODULE（显示加载进度条）
  │            yield return new WaitForSeconds(0.2f)
  ├─ [节点 3] AddComponent() → 挂载各 Manager
  │            JcManager / SoundUtl / UIPoolUtl
  │            IAPManager / AdAndroidManager / AdHelper
  │            FirebaseUtil / UnityAnalyticsManager
  ├─ [节点 4] SDKManager.Instance.Init()
  │            → AdHelper.Init() / FirebaseUtil.Init() / PlatformUtil.Init()
  │            → iOS：初始化 GameSDK（Firebase + Adjust + AdMob）
  ├─ [节点 5] FirebaseUtil.AnalyticsInitialize()
  │            yield return new WaitForSeconds(0.5f)
  └─ InitGame()
        │
        ▼
【InitGame()】
  ├─ LoadDic()         → 协程异步加载字典表（DicMgr.LoadDicAsync）
  └─ LoadDicComplete() → LoadData()
        │
        ▼
【LoadData()】  AppData.LoadData()
  ├─ NewData()            → 实例化所有数据对象（PlayerData / LevelData / MusicData ...）
  ├─ GameData.PreLoadData() → 读取 PlayerPrefs 等本地存档
  └─ LoadDataComplete()
        │
        ▼
【GameEnter()】  协程，游戏正式进入
  ├─ App.inited = true
  ├─ 加载 ComRes1 / ComRes2 UI 包（通用组件）
  ├─ 等待 AppGame.uiCreateComplete（LayerUI 层级创建完毕）
  ├─ CheckUpgradeData() / LoginCheck() / InitDateTime() 等数据检查
  ├─ InitSoundConfig()    → 恢复音乐 / 音效设置
  ├─ 加载 Notice 包       → 打开 NOTICE_MODULE（全局通知层）
  ├─ SendNotice LOADING_RUN_END_AND_WAIT → 进度条跑满动画
  │            yield return new WaitForSeconds(0.3f)
  ├─ [判断分支]
  │   MaxLevel == 1（新用户）
  │     → 加载 Word 包 → 打开 WORD_MODULE（直接进关卡引导）
  │   MaxLevel > 1（老用户）
  │     → 加载 Login 包 → 打开 LOGIN_MODULE（主界面）
  │
  └─ yield return new WaitForSeconds(3f)
       → AdAndroidManager.Initialize()  广告 SDK 延迟初始化
```

---

## 关键文件说明

| 文件 | 职责 |
|---|---|
| `Assets/Scenes/Loading.unity` | 游戏首场景，Build Settings 中 index 0 |
| `Scripts/App_Core/AddGameCore.cs` | 场景入口脚本，运行时动态创建 GameCore / GameManager |
| `Scripts/App_Core/GameCore.cs` | 游戏启动主脚本，继承 GameCoreBase，串联完整初始化流程 |
| `Plugins/AppEngine/Core/GameCoreBase.cs` | 引擎层基类，定义启动模板方法（Config / InitApp / InitGUI 等） |
| `Scripts/App_Core/GameManager.cs` | UI 模块管理器，继承 UIManager，管理所有窗口的打开 / 关闭 / 互斥 |
| `Scripts/App_Core/AppGame.cs` | 全局静态容器，持有各 Manager 引用及 UI 层对象 |
| `Scripts/App_Data/AppData.cs` | 本地数据管理，负责 NewData / LoadData / SaveData |
| `Scripts/APP_SDK/SDKManager.cs` | SDK 统一初始化入口（AdHelper / Firebase / PlatformUtil） |

---

## 关键设计点

| 节点 | 说明 |
|---|---|
| AddGameCore 间接创建 GameCore | 防止场景重载时产生多个实例，且不侵入引擎层基类代码 |
| CoroutineInitLoading 协程分步 | SDK 初始化在进度条就绪后再执行，避免黑屏卡顿 |
| LoadDic → LoadData 严格顺序 | 字典表必须先于数据加载，因为数据解析依赖字典 |
| 等待 uiCreateComplete 标志位 | GameEnter 不提前渲染 UI，确保 LayerUI 层级已建好 |
| 广告 SDK 延迟 3 秒初始化 | 等玩家进入主界面后再初始化，降低启动帧压力 |
| 新老用户分流 | MaxLevel == 1 直接进关卡引导，MaxLevel > 1 进主界面 |

---

## UI 层级结构（AppGame）

GameManager 根据模块配置的 `layer` 字段，将窗口挂载到对应层级：

| 层级名 | 对应容器 | 用途 |
|---|---|---|
| root (0) | `App.stage` | 最底层，场景根节点 |
| ui (1) | `AppGame.uiLayer` | 普通 UI 界面 |
| window (2) | `AppGame.windowLayer` | 弹窗 |
| alert (3) | `AppGame.alertLayer` | 警告 / 确认框 |
| top (4) | `AppGame.topLayer` | 最顶层（飞金币、飞星星等特效） |
