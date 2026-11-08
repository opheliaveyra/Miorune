# 虚拟屏：可行性与真机验证报告

- **版本**：0.1.100-mod（Shizuku 集成在此版）
- **设备**：红米 23076RA4BC / Android 14（API 34）/ 1080×2460；华硕 ASUS_AI2401_A / Android 12（API 31）/ 1080×2400
- **目标**：角色在独立虚拟屏里刷抖音（启动 → 看到画面 → 滑动），不影响主屏

## 结论：整条链路可行，前提是有 ADB 级权限（电脑 adb **或** 手机上的 Shizuku）

### 红米 Android 14（0.1.84，电脑 adb 执行）

| 步骤 | 做法 | 结果 |
|---|---|---|
| 建虚拟屏 | MediaProjection（用户点「立即开始」授权）+ mediaProjection 前台服务 + registerCallback | ✅ displayId=2，720×1280，dpi 320，状态 ON |
| 把抖音放到该屏 | **adb shell** `am start --display 2 -n com.ss.android.ugc.aweme/.splash.SplashActivity` | ✅ 无 SecurityException |
| 读画面 | 探针用 ImageReader 读虚拟屏 Surface | ✅ 720×1280，采样非黑点 9504（整帧有内容） |
| 向该屏滑动 | **adb shell** `input -d 2 swipe 360 1000 360 300 300` | ✅ 滑动前后帧 md5 不同，画面真的变了 |

### 华硕 Android 12（0.1.100，**手机内 Shizuku 执行，不插电脑**）

| 步骤 | 做法 | 结果 |
|---|---|---|
| Shizuku 授权 | 探针「Shizuku 授权」→ 系统弹「要允许Cleos使用 Shizuku吗」→ 始终允许 | ✅ 授权成功 |
| 建虚拟屏 | MediaProjection | ✅ displayId=4，720×1280，dpi 320 |
| 启动抖音到该屏 | Shizuku 执行 `am start --display 4 -n com.ss.android.ugc.aweme/.splash.SplashActivity` | ✅ `dumpsys activity` 显示 **Display #4** 上是 `com.ss.android.ugc.aweme/.splash.SplashActivity`，主屏仍是探针 |
| 读画面 | ImageReader | ✅ 导出帧确认是**抖音首页推荐流**（720×1280） |
| 向该屏滑动 | Shizuku 执行 `input -d 4 swipe 360 1024 360 256 300` | ✅ 退出码 0，**随后画面从"推荐流首页"翻到了另一个直播间**（`@sAmern 脑叶公司`） |

**注意**：抖音如果已经在主屏前台跑着，`am start --display` 只会把 intent 投给已有实例（"Activity not started, intent has been delivered to currently running top-most instance"），不会新建到虚拟屏。**先 `am force-stop com.ss.android.ugc.aweme`** 再启动才稳定。

## 0.1.60 的假设被修正的地方

1. 当时以为「创建虚拟屏不需要权限」——**Android 14 上不对**：需要 `CAPTURE_VIDEO_OUTPUT` 或 **MediaProjection token**。
2. Android 14 还要求：投屏 token 必须运行在 **mediaProjection 类型的前台服务**里，且建屏前必须 **registerCallback**（否则 IllegalStateException）。
3. `am start --display` / `input -d` 需要 ADB 级权限（MANAGE_ACTIVITY_TASKS / INJECT_EVENTS）；**adb shell 与 Shizuku 是同一档**，两者实测都可用。
4. 之前以为「Shizuku 的 IShizukuService AIDL 不在公开包里」——**不对**：`dev.rikka.shizuku:api` 会传递引入 `dev.rikka.shizuku:aidl`，
   里面有 `moe.shizuku.server.IShizukuService`（含 `newProcess`）和 `IRemoteProcess`。0.1.100 就是直接用它跑的命令。

## 探针改动（`ui/probe/`）

- `<queries>` 声明抖音与 Shizuku 包（修包可见性导致的"抖音没装"误报）
- `MediaProjectionService`：mediaProjection 前台服务
- 创建虚拟屏走 MediaProjection（先起服务 → 拿 token → registerCallback → createVirtualDisplay）
- **0.1.100**：注册 `rikka.shizuku.ShizukuProvider`，探针加「Shizuku 授权」按钮（`Shizuku.requestPermission` + 结果回调）；
  「启动抖音」「向该屏滑动」改由 `IShizukuService.newProcess` 以 shell 身份执行 `am/input`。
  去掉了注定失败的 `Instrumentation.sendPointerSync`（主线程调用必抛异常，且事件只落主屏）。

## 未做 / 风险

- 声音输出、发热、抖音反作弊/检测虚拟屏、登录（扫码/验证码）都未验证。
- 主屏与虚拟屏同时使用时，抖音生命周期/降优先级未验证。
- 荣耀（Android 10）没有公开 createVirtualDisplay，无法覆盖。
- Shizuku 服务在 Android 11+ 首次需要电脑 adb 或 root 启动一次；本机实测为已启动状态。

## 下一步建议

1. 生产化：把「角色刷抖音」做成一个正式入口（现在只有验证探针），并处理帧率/内存。
2. 虚拟屏里的抖音登录态（扫码）怎么完成，还没方案。
3. 加"启动前先 force-stop 目标 App"，避免 am start 命中主屏已有实例。
