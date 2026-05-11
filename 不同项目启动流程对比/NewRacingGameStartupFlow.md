# NewRacing 项目启动流程

## 概览

NewRacing 是一款 Unity 音乐节奏赛车游戏，采用多场景架构 + MonoSingleton 管理器体系。整个启动流程可分为 **5 个标准阶段**，由事件驱动的 `GameState` 状态机串联。

---

## 场景结构

| 场景 | 路径 | 职责 |
|------|------|------|
| `LaunchScene` | `Assets/Scenes/LaunchScene.unity` | **入口场景**，负责初始化所有全局系统 |
| `MainScene` | `Assets/Scenes/MainScene.unity` | 主菜单，展示歌曲列表与排行榜 |
| `LoadingScene` | `Assets/Scenes/LoadingScene.unity` | 过渡场景（当前流程主要使用 UI 覆盖方式替代） |
| `GameScene` | `Assets/Scenes/GameScene.unity` | 游戏主场景，实际游玩区域 |

---

## 启动流程总图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          LaunchScene                                │
│                       LaunchScripts.Awake()                         │
│                                                                     │
│  Application.targetFrameRate = 60                                   │
│  DOTween.SetTweensCapacity(1000, 50)                                │
│  IPManager.GetCountryCode()          → 异步获取地区码               │
│  ABTestManager.InitConfig()          → 拉取 Firebase 远程配置       │
│  RecordDataManager.Init()            → 初始化本地存档 / 首次发放金币│
│  RankManager.Init()                  → 异步拉取排行榜数据           │
│                     ↓ (延迟 0.4s)                                   │
│                  ShowLaunchUI()                                      │
│                     ↓                                               │
│         PlayGameCount == 0?                                         │
│        /               \                                            │
│      是(新用户)        否(老用户)                                    │
│       ↓                  ↓                                          │
│  LoadLoadingLevel    LoadMainScene()                                │
│  (guideSongId=1001)       ↓                                         │
│       ↓             ┌─────────────┐                                 │
│       │             │  MainScene  │                                 │
│       │             │  PivotUI    │                                 │
│       │             │  SongListUI │                                 │
│       │             └──────┬──────┘                                 │
│       │                    │ 玩家选曲                               │
│       └────────────────────┘                                        │
│                    ↓                                                │
│          PreloadAndLoadCoroutine()                                  │
└─────────────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    Phase 3 — 资源预加载                              │
│             PreloadAndLoadCoroutine (LevelManager)                  │
│                                                                     │
│  CloseUI(PivotUI / SongListUI)                                      │
│  OpenUI<LoadingUI>                                                  │
│  AudioManager.Clear()              → 清理旧音频缓存                 │
│  ObjectPool.Clear()                → 清理对象池                     │
│  Resources.UnloadUnusedAssets()                                     │
│  GC.Collect()                                                       │
│                                                                     │
│  并行加载:                                                          │
│  ├── BGM (远程/本地)                                                │
│  └── NoteData CSV (首次=教程999, 其余=实际曲目)                     │
│                                                                     │
│  等待 getSong && getNote (最长 10s 超时)                            │
│  GameDataManager.Init(songId)      → 重置分数/血量/连击/星级        │
│  ObjectPool.Preload(block系列)                                      │
│  Shader.WarmupAllShaders()                                          │
│  SceneManager.LoadSceneAsync("GameScene")                           │
│  进度条推进至 100%                                                  │
│  CloseUI<LoadingUI>                                                 │
│  OpenUI<GamePlayUI>                → 显示 "Ready" 界面              │
│  CameraFollow.StartDotweenTransition()  → 开场镜头动画             │
└─────────────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    Phase 4 — GameScene 初始化                        │
│                      GameMainScene.Awake()                          │
│                                                                     │
│  AudioManager.StopPreview()        → 停止菜单预览音乐               │
│  MultiPathConnector.Init()         → 构建道路路径系统               │
│  BlockGenerator.Initialize()       → 排序 NoteData 列表            │
│  BlockGenerator.StartSpawning()                                     │
│  ├── SpawnCoroutine()              → 按音频时间前置生成方块         │
│  ├── RecycleCoroutine()            → 回收已错过的方块               │
│  └── SpawnCoin()                   → 在歌曲末尾位置生成金币         │
│  CarController.Init()              → 初始化玩家赛车控制器           │
│                                                                     │
│  GameState → Ready                                                  │
└─────────────────────────────────────────────────────────────────────┘
                         ↓ 玩家点击屏幕
┌─────────────────────────────────────────────────────────────────────┐
│                    Phase 5 — 游戏进行 & 结算                         │
│                                                                     │
│  GameState → Playing               → BGM 开始播放                  │
│  (首次教程) GuideUI 展示           → 引导步骤 2                     │
│                                                                     │
│  游戏结束流程:                                                      │
│  GameState → GameToResult          → 结束镜头动画                  │
│  GameState → GameResult            → SaveGameResult() + ResultUI   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五个标准阶段

### Phase 1 — 应用启动 (LaunchScene)

**入口文件:** `Assets/Scripts/GameMain/LaunchScripts.cs`

`LaunchScripts` 是游戏唯一的真正入口点，挂载于 `LaunchScene` 中，在 `Awake()` 中依序完成以下工作：

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | `DontDestroyOnLoad(this)` | 跨场景持久化 |
| 2 | `Application.targetFrameRate = 60` | 锁帧 |
| 3 | `DOTween.SetTweensCapacity` | 预分配 Tween 容量 |
| 4 | `IPManager.GetCountryCode()` | 异步地区码，存入 `RecordDataManager` |
| 5 | `ABTestManager.InitConfig()` | 拉取远程配置 |
| 6 | `RecordDataManager.Init()` | 初始化本地存档 |
| 7 | `RankManager.Init()` | 拉取排行榜 |
| 8 | `ShowLaunchUI()` (0.4s 后) | 用户分流 |

---

### Phase 2 — Singleton 基础设施建立

项目使用两种单例模式：

**`Singleton<T>`** (`Assets/Scripts/Common/Singleton.cs`)
- 纯 C# 懒加载线程安全单例
- 使用方：`GameDataManager`、`ABTestManager`、`RankManager`

**`MonoSingleton<T>`** (`Assets/Scripts/Common/MonoSingleton.cs`)
- MonoBehaviour 单例，默认 `DontDestroyOnLoad`
- 若场景中不存在实例，自动创建新 `GameObject`
- 使用方：`LevelManager`、`RecordDataManager`、`AudioManager`、`UIRoot`、`BlockGenerator`、`CameraFollow`、`GameMainScene` 等

---

### Phase 3 — 资源预加载 (LevelManager)

**入口文件:** `Assets/Scripts/Manager/LevelManager.cs`

`LevelManager` 是加载系统的核心，提供两条路径：

| 路径 | 方法 | 目标 |
|------|------|------|
| Path A | `LoadMainScene()` | → `MainScene`（主菜单） |
| Path B | `LoadLoadingLevel(songId)` | → `GameScene`（游戏场景） |

Path B 的核心协程 `PreloadAndLoadCoroutine()` 负责在进入 GameScene 前完成所有资源加载，保证游戏流畅无卡顿。

---

### Phase 4 — 游戏场景初始化 (GameMainScene)

**入口文件:** `Assets/Scripts/GameMain/GameMainScene.cs`

GameScene 加载完成后，`GameMainScene.Awake()` 负责完成道路系统、方块生成器和赛车控制器的初始化，并将 `GameState` 置为 `Ready`。

---

### Phase 5 — 游戏进行与结算

**入口文件:** `Assets/Scripts/UI/GamePlayingUI.cs`

玩家点击屏幕后，`GameState` 切换至 `Playing`，BGM 开始播放，游戏正式运行。

---

## GameState 状态机

**核心文件:** `Assets/Scripts/Manager/GameDataManager.cs`

`GameDataManager` 是全局状态单一数据源，所有系统通过 `Action<GameState> OnGameStateChanged` 事件响应状态变化。

```
Ready ──────────── 玩家点击 ──────────→ Playing
                                           │
                         ┌─────────────────┤
                         ↓                 ↓
                       Paused           HP = 0
                         │                 ↓
                         └──────────→   Revive
                                          │
                              ┌───────────┤
                              ↓           ↓
                         [放弃]      ReviveSuccess
                              │           │
                              │           ↓
                              │         Ready（界面）
                              │           ↓ 继续游戏
                              │         Playing
                              ↓
                       歌曲结束 → GameToResult → GameResult
```

| 状态 | 触发时机 | 系统响应 |
|------|----------|----------|
| `Ready` | 场景加载完 / 复活后 | 显示 Ready 界面 |
| `Playing` | 玩家点击 / 复活后继续 | BGM 播放，方块生成开始计时 |
| `Paused` | 点击暂停按钮 | `Time.timeScale = 0`，BGM 暂停，显示 PauseUI |
| `Revive` | HP 归零 | 显示 ReviveUI（广告复活） |
| `ReviveSuccess` | 选择复活 | 回到 Ready 状态显示 |
| `GameToResult` | 歌曲播放完毕 | 触发结束镜头动画 |
| `GameResult` | 镜头动画结束 | `SaveGameResult()`，显示 ResultUI |

---

## 关键管理器初始化顺序

| 顺序 | 管理器 | 类型 | 初始化时机 | 职责 |
|------|--------|------|-----------|------|
| 1 | `LaunchScripts` | `MonoBehaviour` | LaunchScene `Awake` | 应用入口，调用所有其他初始化 |
| 2 | `ABTestManager` | `Singleton<T>` | LaunchScripts.Awake | Firebase 远程配置（MaxHp、解锁费用等） |
| 3 | `RecordDataManager` | `MonoSingleton<T>` | LaunchScripts.Awake | 加密 PlayerPrefs 封装，首次发放金币 |
| 4 | `RankManager` | `Singleton<T>` | LaunchScripts.Awake | 从服务器拉取排行榜数据 |
| 5 | `LevelManager` | `MonoSingleton<T>` | 懒加载（首次访问） | 场景加载 / 预加载协程 |
| 6 | `UIRoot` | `MonoSingleton<T>` | 懒加载 | Canvas / UI 生命周期管理 |
| 7 | `AudioManager` | `MonoSingleton<T>` | 懒加载 | BGM / SFX / 预览音频 |
| 8 | `GameDataManager` | `Singleton<T>` | `PreloadAndLoadCoroutine` 中调用 `.Init()` | 分数/血量/连击/状态、谱面数据缓存 |
| 9 | `ObjectPool` | `MonoSingleton<T>` | `PreloadAndLoadCoroutine` 预热 | 可复用方块/金币/特效对象池 |
| 10 | `BlockGenerator` | `MonoSingleton<T>` | GameScene `Awake`（由 GameMainScene 调用） | 生成/回收游戏方块 |
| 11 | `CameraFollow` | `MonoSingleton<T>` | GameScene 挂载，随场景加载 | 相机跟随 + 开场镜头过渡 |
| 12 | `SceneColorManager` | `MonoSingleton<T>` | `GameDataManager.Init` 时重置 | 按节拍时间触发道路颜色变换事件 |

---

## 架构要点

- **无独立 Bootstrap 场景**：`LaunchScene` 中的 `LaunchScripts` 本身即承担 Bootstrap 职责。
- **`DontDestroyOnLoad` 覆盖全局**：`LaunchScripts` 及所有 `MonoSingleton` 实例均跨场景持久化。
- **`LoadingScene.unity` 为遗留资产**：当前主加载路径通过 UI 覆盖（`LoadingUI`）实现，不切换至独立 Loading 场景。
- **`GameState` 是单一事实来源**：几乎所有子系统均订阅 `GameDataManager.OnGameStateChanged` 事件来响应状态切换。
- **新老用户分流**：通过 `RecordDataManager.PlayGameCount` 判断，新用户直接进入教程关卡（歌曲 ID 1001），老用户落在主菜单。

---

## 相关源文件速查

| 文件 | 职责 |
|------|------|
| `Assets/Scripts/GameMain/LaunchScripts.cs` | 应用入口 |
| `Assets/Scripts/GameMain/GameMainScene.cs` | GameScene 初始化 |
| `Assets/Scripts/Manager/LevelManager.cs` | 场景加载 / 预加载协程 |
| `Assets/Scripts/Manager/GameDataManager.cs` | 全局状态机与游戏数据 |
| `Assets/Scripts/Manager/AudioManager.cs` | 音频管理 |
| `Assets/Scripts/Manager/RankManager.cs` | 排行榜 |
| `Assets/Scripts/Manager/ABTestManager.cs` | A/B 测试 / 远程配置 |
| `Assets/Scripts/Manager/RecordDataManager.cs` | 本地存档 |
| `Assets/Scripts/GameMain/BlockGenerator.cs` | 方块生成与回收 |
| `Assets/Scripts/Common/Singleton.cs` | 纯 C# 单例基类 |
| `Assets/Scripts/Common/MonoSingleton.cs` | MonoBehaviour 单例基类 |
| `Assets/Scripts/UI/GamePlayingUI.cs` | Ready/Playing UI 控制 |
| `Assets/Scripts/GameMain/CarController.cs` | 玩家赛车控制器 |
| `Assets/Scripts/GameMain/MultiPathConnector.cs` | 多路径道路系统 |
