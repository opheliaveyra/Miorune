# Cleos 真机复测 · 终版汇总

> 执行人：津渡（DSH）
> 汇总日期：2026-11-07
> 复测周期：2026-10-28 → 2026-11-07
> 目标仓库：opheliaveyra/Cleos-Unofficial-Mod
> 目标分支：feature/chat-file-upload-and-hold-to-talk
> 详细过程：见 [RETEST-ACCEPTANCE-20261028.md](RETEST-ACCEPTANCE-20261028.md) 与 [RETEST-ACCEPTANCE-20261028-SUPPLEMENT.md](RETEST-ACCEPTANCE-20261028-SUPPLEMENT.md)

---

## 一、结论速览

清单共 **20 项**：

| 状态 | 数量 |
|---|---|
| ✅ 通过（含观察项） | **14** |
| ⚠️ 部分通过 | **2** |
| ❌ 阻塞 / 未执行 | **3** |
| — 不适用 | **1** |

**最重要的一条**：P0 提交 `42fc821` 补的三处后台 Token 用量统计（Recap / FeedAi / Letters）**全部实测生效**，`token_usage` 表分别出现 `purpose=recap` 2177、`letter` 2142、`feed` 781 的真实记录。

---

## 二、逐项结果

### 一、HTML 小组件状态恢复（Issue #3）

| 编号 | 项目 | 结果 | 关键证据 |
|---|---|---|---|
| HTML-01 | 输入框内容恢复 | ✅ 通过 | 输入 `RETEST-HTML-01` 后关闭重开，内容保留 |
| HTML-02 | 滚动位置恢复 | ✅ 通过 | 长 HTML 滚到第 10 段后收起重开，停在原位 |
| HTML-03 | 杀进程后恢复 | ✅ 通过 | `am force-stop` 后重启，输入内容仍在 |
| HTML-04 | 「重置状态」按钮 | ✅ 通过 | 重置后重开恢复为初始占位符 |
| HTML-05 | 多个组件状态隔离 | ⚠️ 部分 | 组件能生成/渲染/输入/回显，但 A/B 隔离未实测（见下） |

**HTML-05 说明**：两个组件的持久化行为不同 —— 输入框组件用了 localStorage（状态保留），计数器组件只用内存变量（收起后归零）。这是 TA 生成 HTML 时是否主动写存储的差异，不是 LiveWidget 的 Bug。要实测隔离需两个都用 localStorage 的组件作对照，本轮未完成。

### 二、五子棋（Issue #2）

| 编号 | 项目 | 结果 | 关键证据 |
|---|---|---|---|
| GMK-01 | 旋转屏幕 | ✅ 通过 | 3黑3白局面旋转：棋子/高亮/回合全保留，转回不崩不丢，旋转后可继续对局（4黑4白） |
| GMK-02 | 断网失败处理 | ✅ 通过 | 飞行模式落子 → 提示「找不到服务器」→ 棋子保留 → 无自动重试 → 恢复网络手动重试成功、只落一子 |
| GMK-03 | 超时（60s） | ❌ 阻塞 | 需「地址不变但不可达」的网络层手段，设备无 root 做不到 |
| GMK-04 | 存档隔离 | ✅ 通过 | 五子棋与井字棋存档独立；重开五子棋不影响井字棋 |
| GMK-05 | 角色删除 / ID 复用 | ✅ 观察完成 | 删 TA 后静默回引导页、无报错、棋局被清、选择列表正确移除 |
| GMK-06 | 棋局是否进 ZIP 备份 | ✅ 观察完成 | ZIP 内仅 `backup.json`，无任何 game/gomoku/tic 字段 —— 棋局不进备份 |

### 三、Token 用量标签（P0 / commit 42fc821）

| 编号 | 项目 | 结果 | 关键证据 |
|---|---|---|---|
| TOK-00 | 基线记录 | ✅ 完成 | 34 次请求 / 187892 tokens，用途仅 chat/game_* |
| TOK-01 | purpose=recap | ✅ 通过 | **recap 2177**（`04:30 1600+577`）|
| TOK-02 | purpose=feed | ✅ 通过 | **feed 781**（`04:34 720+61`）|
| TOK-03 | purpose=letter | ✅ 通过 | **letter 2142**（`04:27 1941+201`）|

复核总量：57 次请求 / 368083 tokens。每条记录 scope=TA、模型名、输入/输出 token 均为服务商返回的实际值。附带验证了「服务商没返回 usage 时标『未提供』，不拿 0 冒充」的设计（实际出现 1 次）。

### 四、可选 PR 分支

| 编号 | 项目 | 结果 | 原因 |
|---|---|---|---|
| PR-01 | PR #5 紫雾手账截图矩阵 | ❌ 未执行 | 需 checkout `feat/lavender-scrapbook-batch01` 单独构建 |
| PR-02 | PR #4 纸绘图标截图 | ❌ 未执行 | 需 checkout `feat/paper-functional-icons` 单独构建 |

### 五、PR #11 回归（lint 修复）

| 编号 | 项目 | 结果 | 关键证据 |
|---|---|---|---|
| LINT-01 | 启动屏背景 | — 不适用 | 设备 Android 10（SDK 29），启动屏属性需 API 31+ |
| LINT-02 | TA 设置页旋转 | ✅ 部分 | 设置项状态保持、不崩溃；但页面锁定竖屏，系统旋转不生效 |
| LINT-03 | 小组件放大弹窗 | ✅ 通过 | 弹窗居中、约 80% 宽、可滚动、两次开关正常 |

---

## 三、五个值得记的发现

### 1. P0 的三处 usageTag 全部生效

`42fc821` 给 Recap / FeedAi / Letters 补的 usageTag 在真机上产生了实际记录，链路完整：

```
stream(usageTag) → onUsage 回调 → TokenUsageEntity → UsagePage 按 purpose 展示
```

### 2. Key 按服务商地址绑定

服务商页原文：「Key 按地址存，换型号不用重填」。

实测：把接口地址从 `https://speed31.toter.me` 改成别的地址后，**立即**提示「TA 还没填 API Key」—— 本地校验先失败，请求根本没发出去。所以「改地址」无法用来模拟网络超时（这是 GMK-03 做不了的原因）。

### 3. 五子棋横屏时顶部信息栏不可见

横屏下棋盘放大填满屏幕，标题 / TA 名字 / 回合文字被挤出可视区。棋盘本身、棋子、高亮都正常。

是否算缺陷需产品判断：棋盘优先可以理解，但丢失「轮到谁」对可用性有影响。

### 4. 删除 TA 会静默清掉棋局，且没有任何提示

删除确认框列了对话、信、记忆、收藏、日记，**唯独没提棋局**。删除后进五子棋直接回到「邀请 TA 下一局」，没有报错也没有说明。

而且五子棋只存「最近一局」—— 那局属于被删的 TA，于是其他 TA 的旧局也一起没了。

### 5. HTML 组件的持久化取决于 TA 写的 JS

同一个 LiveWidget 机制下：

| 组件 | 是否用 localStorage | 重开后 |
|---|---|---|
| 输入框组件 | 是 | 内容保留 |
| 计数器组件 | 否（内存变量） | 归零 |

App 侧按 HTML 内容 SHA 生成存储 key，机制本身正常；存不存由 TA 生成的代码决定。

---

## 四、构建与交付信息

| 项 | 值 |
|---|---|
| 构建基线 | `8f2b237`（Merge PR #11，含 Android CI 与 5 个 lint 修复）|
| 构建命令 | `:app:assembleRelease` |
| 构建结果 | BUILD SUCCESSFUL（约 2-5 分钟增量） |
| APK 版本 | versionCode 300 / 0.1.92-mod-retest（临时，测试后已还原为 264 / 0.1.92-mod）|
| 设备 | HONOR YAL-AL50，Android 10（SDK 29），1080×2340 @480dpi |
| 测试 TA | Mira（60 条消息）、Echo（19 条）；Nova 用于 GMK-05 已被删除 |
| 模型 | 入梦 Flash · 肖恩（speed31.toter.me） |

### 相关 commit

| SHA | 内容 |
|---|---|
| `42fc821` | P0：补齐 Recap/FeedAi/Letters 后台 Token 用量统计 |
| `60514ee` | P0 真机验证报告与截图 |
| `e9aa865` | P1/P2 缺口审计 |
| `8933c22` | 最终功能验收报告（首轮） |
| `1122c60` | 复测报告（无 Key 阶段） |
| `66c68d4` | Key 配置后补充验证 |
| `f4828ac` | TOK-01 触发尝试记录 |
| `32d696a` | P0 三项 usageTag 实测通过 |
| `50e37eb` | 测试 TA 命名 |
| `b2e03b7` | GMK-01 完整通过 + HTML-05 组件验证 |
| `a0bbff7e` | GMK-05 角色删除观察 |
| `e6f4e48` | HTML-05 组件 B 与持久化发现 |
| （本次） | 终版汇总 + GMK-05 章节补回 |

---

## 五、未验证项与原因

| 项 | 原因 | 需要什么才能做 |
|---|---|---|
| GMK-03 60s 超时 | 改地址会让 Key 失效（本地校验先失败）；断网是「找不到服务器」不是超时 | 保持地址不变但让该地址不可达（hosts 劫持 / 路由器屏蔽 / 代理拦截）；或预置一个「同地址但不响应」的中转 |
| PR-01 / PR-02 | 需 checkout 对应分支并重新构建安装 | 分支切换 + 一次完整构建（约 5 分钟） |
| LINT-01 启动屏 | 设备 Android 10，启动屏属性需 API 31+ | Android 12+ 设备 |
| HTML-05 A/B 隔离 | 需要两个都用 localStorage 的组件作对照 | 再让 TA 生成一个用 localStorage 的组件 |

---

## 六、建议

### 对代码

1. **五子棋横屏布局**：如果认为回合信息在横屏下必须可见，可改为信息栏固定顶部 + 棋盘填充剩余空间。
2. **删除 TA 时的棋局提示**：既然棋局会被一并清掉，确认框里加一句「棋局存档也会一起清掉」会更诚实。
3. **HTML 组件持久化**：不是缺陷，但若希望「小玩意儿」天然支持状态保持，可以在 Prompt 里建议 TA 用 localStorage（现在只在 ToolGroup.Live 的说明里提到不联网，没提存储）。

### 对测试

1. GMK-03 与 PR-01/02 是唯一还值得补的项，其余项结论已经稳定。
2. 真机 adb 自动化的坑已整理在补充报告的「自动化限制」一节（输入法、坐标漂移、Compose 焦点等），后续可复用。

---

## 七、证据索引

全部截图与日志在 `docs/verification/`，共 61 个文件（55 张截图 + 报告与清单）。按章节：

| 章节 | 证据文件 |
|---|---|
| HTML-01~04 | `html01-input-restored.png`、`html02-scroll-restored.png`、`html02-scrolled-before.png`、`html03-kill-restored.png`、`html04-before-reset.png`、`html04-after-reset.png` |
| HTML-05 | `html05-widget-a-*.png`、`html05-widget-b-*.png`、`html05-widget-a-echo-proof.txt` |
| GMK-01 | `gmk01-full-before-rotate.png`、`gmk01-full-landscape-3b3w.png`、`gmk01-full-portrait-restored.png`、`gmk01-landscape.png`、`gmk01-rotate-restored.png` |
| GMK-02 | `gmk02-airplane-on.png`、`gmk02-network-fail.png`、`gmk02-retry-success.png`、`gmk-api-key-missing.png` |
| GMK-04 | `gmk04-save-persisted.png`、`gmk04-tic-save.png`、`gmk04-tic-after-move.png`、`gmk04-gomoku-independent.png`、`gmk04-restart-isolated.png` |
| GMK-05 | `gmk05-nova-save-created.png`、`gmk05-delete-confirm-dialog.png`、`gmk05-after-delete-home.png`、`gmk05-after-delete-gomoku.png`、`gmk05-picker-no-nova.png` |
| GMK-06 | `gmk06-zip-contents.txt` |
| TOK | `tok00-baseline.png`、`tok-chat-record-verified.png`、`tok01-tok03-recap-letter-verified.png`、`tok01-tok03-recent-requests-detail.png`、`tok02-feed-post-created.png`、`tok02-feed-verified-final.png` |
| LINT | `lint02-before-rotate.png`、`lint02-state-persisted.png`、`lint03-expand-dialog.png` |
| 阻塞证据 | `blocker-no-api-key.png` |
| TA 命名 | `retest-ta-naming.png` |
