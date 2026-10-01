# OAS 自定义优化与修复记录 (Changelog)

本项目基于原版 [runhey/OnmyojiAutoScript](https://github.com/runhey/OnmyojiAutoScript) 进行二次调优与稳定性增强，专为解决多账号挂机、长时间空闲断连、UI 路径死循环及部分特定副本的异常中断而设计。

---

## [2026-10-01] 核心稳定性补丁与任务调优

### 1. 设备连接与长时间挂机保活 (Device & Connection Resilience)
- **[module/device/method/utils.py](file:///e:/omyoji/OAS/module/device/method/utils.py)** (`handle_adb_error`):
  - **问题**：原代码仅白名单匹配 6 种特定的英文字符串。在多任务间隙挂机超过 1 小时后，模拟器或宿主机端口转发断开，ADB 抛出空信息或一般性 `AdbError`，被误判为不可恢复错误而直接返回 `False`，导致 5 次重试机会被立即掐断（`break`），任务进程直接崩溃退出。
  - **修复**：放宽异常识别，当遇到一般性 `AdbError` 时在日志中记录并返回 `True`，允许系统按预设策略进行 5 次渐进式延迟重试与自动重连。
- **[module/device/connection.py](file:///e:/omyoji/OAS/module/device/connection.py)** (`adb_disconnect`):
  - **问题**：在断开并重连 ADB 时，原代码仅清理了 `hermit`、`droidcast`、`minitouch` 的会话，未清理 `u2` 的属性缓存，导致即使 ADB 物理重连成功，`self.u2` 依然持有失效的旧 HTTP 会话，重试时继续抛出连接异常。
  - **修复**：在断开操作中增加 `del_cached_property(self, 'u2')`，确保重连后重新初始化 ATX-agent 端口映射与会话对象。
- **[module/device/method/uiautomator_2.py](file:///e:/omyoji/OAS/module/device/method/uiautomator_2.py)** (`retry_wrapper`):
  - **问题**：uiautomator2 发生底层网络超时或连接中断时，异常未能有效触发 ADB 重连。
  - **修复**：在重试装饰器中显式捕获 `requests.ConnectionError` 和 `requests.Timeout`，自动执行 `self.adb_reconnect()` 触发重连与会话刷新。

---

### 2. UI 路由导航与死循环防护 (Game UI & Navigation)
- **[tasks/GameUi/game_ui.py](file:///e:/omyoji/OAS/tasks/GameUi/game_ui.py)** (`ui_goto`):
  - **问题**：当游戏处于无通路页面（例如留在 `page_login` 适龄提示、账号切换界面或未适配弹窗）时，`ui_goto` 在 `not path` 分支调用 `ui_get_current_page(skip_first_screenshot=True)`。由于传入了 `True`，系统**完全不截取新画面**，且循环内无任何延时，导致以每秒 25~30 次的高频拿内存中的同一张旧图死循环比对，硬生生把 60 秒耗光后抛出 `Cannot goto page, timeout reached` 异常崩溃。
  - **修复**：找不到路径时，强制截取最新画面（`skip_first_screenshot=False`），增加 `0.5s` 延时避免 CPU 满载，并在 3 秒无进展后主动调用 `try_close_unknown_page` 尝试关闭可能存在的弹窗。
- **[tasks/GameUi/assets.py](file:///e:/omyoji/OAS/tasks/GameUi/assets.py)** / **[tasks/GameUi/page.py](file:///e:/omyoji/OAS/tasks/GameUi/page.py)** / **[tasks/GameUi/additional/additional_clean_view_back.png](file:///e:/omyoji/OAS/tasks/GameUi/additional/additional_clean_view_back.png)**:
  - **问题**：若游戏意外进入“庭院展示/全景模式”（无任何 UI 按钮的纯净视角），脚本因匹配不到庭院特征而无限卡住。
  - **修复**：截取并注册全景模式左上角返回按钮 `I_CLEAN_VIEW_BACK`，将其加入庭院附加检查与全局弹窗关闭清单，实现自动识别并退出全景模式。

---

### 3. 系统更新安全与应急调度 (Updater & Scheduler)
- **[module/server/updater.py](file:///e:/omyoji/OAS/module/server/updater.py)**:
  - **问题**：在 WebUI 上点击“手动更新”或触发自动更新时，若本地存在任何修改过的文件，`git pull` 会报冲突失败或导致本地优化被覆盖。
  - **修复**：在拉取前后自动嵌入 `git stash` 与 `git stash pop`，实现带本地修改的安全平滑更新。
- **[module/config/config.py](file:///e:/omyoji/OAS/module/config/config.py)**:
  - **问题**：当某个任务连续失败触发应急重启（Restart）时，若此时存在其它已到期的任务，Restart 可能被排在队尾无法第一时间执行救急。
  - **修复**：优化应急重启任务的调度抢占逻辑，确保 Restart 拥有最高执行优先级，迅速重启游戏恢复运行。

---

### 4. 任务业务逻辑深度优化 (Task Enhancements)
- **[tasks/KekkaiUtilize/script_task.py](file:///e:/omyoji/OAS/tasks/KekkaiUtilize/script_task.py)** (结界寄养/提取):
  - **问题**：经验盒已满或界面识别异常时，脚本会陷入反复点击“提取”按钮的无限循环。
  - **修复**：增加经验提取数值阈值判定，并加入最大连续点击保护上限，超出限制后安全跳过，防止卡死。
- **[tasks/DemonEncounter/script_task.py](file:///e:/omyoji/OAS/tasks/DemonEncounter/script_task.py)** (逢魔之时):
  - **问题**：切换预设阵容时，小键盘数字与队伍配置序号的几何映射计算存在偏移，导致阵容切换错误。
  - **修复**：修正队伍编号坐标计算映射算法，确保准确选定预设队伍。
- **[tasks/Component/SwitchSoul/switch_soul.py](file:///e:/omyoji/OAS/tasks/Component/SwitchSoul/switch_soul.py)** (御魂切换):
  - **问题**：从庭院前往式神录换御魂时，若右下角卷轴处于收起状态，会因找不到“式神录”入口而超时。
  - **修复**：前置增加卷轴状态检测，检测到卷轴收起时先自动点击展开卷轴，再进入式神录。
- **[tasks/ActivityShikigami/script_task.py](file:///e:/omyoji/OAS/tasks/ActivityShikigami/script_task.py)** (活动爬塔):
  - **问题**：大型活动爬塔中卡在跳过战斗确认弹窗时，因超时导致整个活动任务判定失败。
  - **修复**：增加跳过弹窗的专用超时重试与兜底点击确认机制。
- **[tasks/FrogBoss/script_task.py](file:///e:/omyoji/OAS/tasks/FrogBoss/script_task.py)** (呱太来袭):
  - **问题**：夜间战斗结算时背景亮度变暗，默认的日间结算图标匹配阈值无法通过。
  - **修复**：新增夜间暗色背景结算图标 `S_WIN_NIGHT` 兜底匹配，解决夜间结算卡住问题。
