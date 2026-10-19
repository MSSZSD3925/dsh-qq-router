# dsh-qq-router

用 QQ 远程驾驶 **本 profile 里已经存在的会话**。

和 `dsh-1bot` 的区别（这是本插件存在的理由）：

| | dsh-1bot | dsh-qq-router |
|---|---|---|
| 跑在哪个 profile | 独立的 `onebot` profile | **desktop profile（同一个进程）** |
| QQ 消息的落点 | 新建 `onebot-private-<QQ>` 会话 | 你指定的**已有会话** |
| 能不能接着做一半的任务 | 不能（会话是另一个进程的） | **能**：注入成 `source.kind='user'` 的普通轮次，等于你在电脑前打字 |
| 会话可见性 | 藏在 `sessions-hidden`（跨进程写会损坏日志） | 就是你平时的会话，UI 里照常显示 |

## 为什么必须跑在同一个 profile

一份会话日志只能有一个写入者。两个进程同时写会损坏 seq。所以「往已有会话里插话」这件事，只能由**持有那个会话的进程**来做 —— 也就是 desktop profile。

## 命令（在 QQ 里，前缀默认 `#`）

```
#list              列出会话与路由编号
#set <路由>        设定默认会话，写进状态文件，重启 DSH 也不丢
#<路由> 内容       把内容注入指定会话
#内容              注入默认会话
#running           列出正在运行的会话（正在生成文字或执行任务的）
#msg [路由]        该会话最新的一条助手回复
#last [路由]       该会话里你自己发的最后一条消息
#status            连接状态与当前路由、待回话轮次
#help              用法
```

`#use` 是 `#set` 的同义写法，保留兼容。

## 路由怎么写

三种形式，稳定性依次递增：

| 形式 | 例子 | 说明 |
| --- | --- | --- |
| 编号 | `#2` | 来自最近一次 `#list`，**按最后活动时间排序，会变**。子会话也会占号 |
| 别名 | `#设定` | 配置里 `aliases` 定义的固定名字 |
| 短 id / 完整 id | `#0300a524` | 最稳，不受活动时间影响 |

因为编号会随时间漂移（你正在聊的会话未必在第一位），长期使用建议 `#set` 固定一个默认会话，或者配别名。

## 回复怎么回到 QQ

模型回复 QQ 的唯一途径是调用工具 `onebot_send`（`target` 形如 `private:<QQ>`）。插件**不自动转发模型正文**，避免你在电脑前同时打字时串话。

补充规则：注入的那一轮结束时，如果模型**没有**调用过 `onebot_send`，插件会把该会话最新的一条助手回复自动发回 QQ（读不到就回一行诊断）。这条是为了避免"任务在跑但你手机上什么都收不到"。

## 状态文件

`#set` 的默认会话存在 `$DSH_HOME/dsh-qq-router-state.json`：

```json
{ "sticky": { "123456789": "session-xxxxxxxx-..." } }
```

删掉这个文件即回到"没有默认会话"的状态。它只记默认路由，不碰任何会话日志。

## 配置

配置写在 `$DSH_HOME/profiles/desktop/cordis.patch.yml` 里本插件的挂载行上。字段见 `lib/index.js` 的 `Config`。

**挂载行 config 的键必须与 `Config` schema 完全一致**：多出任何键，严格校验会让插件加载失败，而失败只写在启动 DSH 的终端里（不落盘），现象是「装好了但一点反应都没有」。改 schema 时同步改 patch。

## 排查

插件在 `$DSH_HOME/dsh-qq-router.log` 里逐条记录：

```
[apply]          插件是否加载（没有这行 = 根本没加载）
[connect]        与 NapCat 正向 WS 的连接结果
[in]             收到的 QQ 消息（含被忽略的原因）
[inject]         注入到哪个会话
[send]           发回 QQ 的消息
[latest]         msg/last 读不到正文时的诊断（events/msgs/roles）
[running]        #running 取不到会话 id 时的诊断
[state]          默认路由的恢复与写入
```

### 两条环境约束

**一、会话正文的取值顺序必须是 `ev.data.message ?? ev.data`。** 事件负载结构不对称：`assistant/message` 的正文在 `data.message` 里，`user/message` 的正文直接就是 `data`（没有再包一层 `message`）。校验点：`#last` 能返回用户自己发的原话。见 `readLatest()`。

**二、`reconnection.attempts` 决定 NapCat 不可用时重试多久。** 正向 WS 连不上时按 5 秒间隔重试该次数后放弃。NapCat 登录异常时先修 NapCat，再重启 DSH 让插件重新连接。

**三、从计划任务启动 GUI 应用（桌面端）的正确写法。** 任务动作直接指向 exe 并显式设置 `WorkingDirectory`（例如 `D:\DSH`）。三个必须避开的坑：

- 不要用 `explorer.exe "路径"` 代启动 —— 实测返回 1 且不产生进程。
- 由 `run-hidden.py` 拉起的进程在 Job Object 里（kill-on-close），从它内部 spawn 的 GUI 进程会随启动器退出被一起杀掉；要启动 GUI 就走 `schtasks /run /TN <任务名>`，交给任务计划服务。
- 目标应用若有 `requestSingleInstanceLock`（DSH 桌面端有），重复启动只会切窗口并立即退出，可安全地用它验证"任务能否拉起 exe"：看启动瞬间进程数峰值是否高于基线。

**四、`#start` 启动 DSH 时必须先等旧实例退净。** 从右键任务栏退出到进程真正消失有几秒，新实例抢在这几秒内启动会拿不到 Electron 单实例锁，250 毫秒内自行退出（无崩溃报告、日志里只有退出直方图）。做法：等进程数归零再启动，且每次启动后若"进程起了又退"就重试，最多三轮。校验点：`~/.dsh/dsh-waker.log` 里出现 `第 N 轮：进程起了又退，准备重试` 后紧接着 `端口已通`。

---

## 远程唤醒（`qq_waker`）

插件跑在 DSH 里，DSH 关着时它不存在，所以"用 QQ 命令启动 DSH"由独立进程 `waker.mjs` 完成。插件只负责安装与查询：

| action | 作用 |
| --- | --- |
| `install` | 写 `waker.config.json` 与隐藏启动脚本；注册 `DSH Waker`（登录自启、常驻、无时限）与 `DSH Start`（直接跑 exe、工作目录、允许多实例）；在启动文件夹建 DSH 快捷方式；立即启动唤醒器 |
| `status` | 两个任务的状态与上次结果、登录自启是否已设、`dsh_port` 是否在听 |
| `start` / `stop` | 启停唤醒器 |
| `uninstall` | 删除两个任务、登录自启快捷方式、隐藏启动脚本（保留配置文件） |

唤醒器的行为：连 NapCat 的 3001，只认白名单 QQ 的 `#start` / `#启动` / `#开机`；先探 `dsh_port`，通了就保持沉默交给插件回答（避免双回复），没通才清理陈旧锁、等旧实例退净、经 `DSH Start` 启动并轮询端口。

日志：`~/.dsh/dsh-waker.log`。
