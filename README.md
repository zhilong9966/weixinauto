# WeixinAuto

Windows 微信 PC 端自动化 SDK。发消息、发文件、监听新消息，对外只需关心一个类：`Wechat`。

```bash
pip install weixinautox4 -i https://pypi.tuna.tsinghua.edu.cn/simple
```

| | |
|---|---|
| **包名 / 导入名** | `weixinautox4` / `weixinauto` |
| **运行环境** | Windows 10/11 + 微信 PC 版 **4.1.15.13**（`weixin.exe`），需保持登录、前台可见 |
| **Python** | **仅 3.11 (Win64)** —— 发布的是 Cython 编译的 cp311 平台 wheel |
| **授权** | 商业授权，一机一码，**300 元/年**，购买后不支持退款，[见文末](#联系我们) |

本文档就是完整的 API 参考，一页看完。第一次用请务必先看 [第 0 节](#0-开始之前先让微信可被读取)，
那是最容易卡住的一步。

---

## 0. 开始之前：先让微信「可被读取」

新版微信是 Qt 架构，**只有当系统检测到「辅助技术」正在运行时，它才会构建可被读取的
无障碍（UIA）树**。没有这棵树，本 SDK 什么都读不到——这是初次使用最容易卡住的一步，
**请务必先完成本节**。

### 做法：登录微信时打开「讲述人」

1. 按 **`Win` + `Ctrl` + `Enter`** 打开 Windows 讲述人（Narrator）
2. **保持讲述人开启的状态下登录微信**
3. 登录成功后再运行本 SDK——**只要能读到消息内容，就说明无障碍树已经建立，可以正常使用了**

> 讲述人会朗读界面，比较吵。登录完、确认能读到消息之后，可以用同样的快捷键关掉试试；
> 若关掉后读不到消息，就保持开启。

### 没生效时会看到什么

日志里出现这一行，就是无障碍树没建立：

```
检测到微信 Qt51514QWindowIcon 外壳，但未发现 mmui::MainWindow。
```

紧接着会抛出：

```
RuntimeError: 检测到微信外壳窗口，但 UIA 主窗未启用。
请在当前窗口完成登录（或关闭微信重开）后，再点击一次启动。
```

**处理办法**：打开讲述人 → **完全退出微信**（不是最小化）→ 重新登录 → 再启动程序。

### SDK 自己也会尝试，但不保证够用

初始化时 SDK 会自动做三件事，让系统以为有辅助技术在运行：

- 写入用户环境变量 `QT_ACCESSIBILITY=1` 并广播 `WM_SETTINGCHANGE`
- 打开 Windows 全局屏幕阅读器标志（`SPI_SETSCREENREADER`）
- 注册 UIA 事件订阅（内部称「弱脉冲」）

这些在多数机器上有效，但**登录阶段常常不够**——所以仍然建议按上面的步骤，
登录时手动开一次讲述人，这是目前最可靠的方式。

---

## 1. 安装

### 国内镜像（推荐）

直连 PyPI 在国内经常超时，用镜像源装：

```bash
pip install weixinautox4==2.8.12 -i https://pypi.tuna.tsinghua.edu.cn/simple
```

换成其它镜像同理，任选其一：

| 镜像 | 地址 |
|---|---|
| 清华大学（推荐） | `https://pypi.tuna.tsinghua.edu.cn/simple` |
| 阿里云 | `https://mirrors.aliyun.com/pypi/simple` |
| 腾讯云 | `https://mirrors.cloud.tencent.com/pypi/simple` |
| 华为云 | `https://repo.huaweicloud.com/repository/pypi/simple` |

嫌每次都带 `-i` 麻烦，把镜像设成默认（只需设一次）：

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

> 镜像是 PyPI 的只读副本，新版本发布后**自动同步**，通常几分钟到几小时。
> 刚发布就装不到最新版是正常的，等一会儿，或临时加 `-i https://pypi.org/simple` 走官方源。

### 直连官方源

```bash
pip install weixinautox4==2.8.12
```

### 固定版本

集成到你自己的项目时写进 `requirements.txt`，**建议锁死版本**，避免上游更新影响生产：

```
weixinautox4==2.8.12
```

> ⚠️ **本包是 cp311 平台专用 wheel**（Cython 编译产物），**只能装在
> Python 3.11 + 64 位 Windows** 上。其它 Python 版本会报
> `No matching distribution found`——这不是网络问题，是没有对应版本的包。
> 部署脚本里请把 Python 版本卡死在 3.11。

导入（包名是 `weixinautox4`，导入名是 `weixinauto`）：

```python
from weixinauto import Wechat, ChatMessage, MessageType
```

查看已安装版本：`weixinauto.__version__`

---

## 2. 快速开始

```python
from weixinauto import Wechat

wx = Wechat()

# 首次使用必须激活（一机一码）
ok, info = wx.activate("LIC-XXXXXXXX-XXXXXXXX")
if not ok:
    raise RuntimeError(f"激活失败：{info}")

# 发送文本
wx.send_message("工作群", "大家好，我是自动化助手")

# 发送文件 / 图片
wx.send_file("工作群", r"E:\1.jpg")
```

> 所有需要微信驱动的方法，在**未激活**时会被自动拦截，只打印一条 warning 日志并返回失败值（`False`/`None`/`{}`），不会抛异常中断你的程序。

---

## 3. 激活与状态

| 方法 | 返回 | 说明 |
|---|---|---|
| `activate(license_key, *, current_version=None, package_name=None, timeout=8.0)` | `tuple[bool, dict\|None]` | 联网激活。**务必判断返回的第一个值**，失败要停下 |
| `is_activated()` | `bool` | 纯本地判断是否已激活，不联网 |
| `ensure_activated()` | `None` | 未激活则抛 `LicenseNotActivatedError` |
| `query_license_status(*, timeout=8.0)` | `dict\|None` | 按本机设备指纹联网查授权状态，不依赖本地文件、不需激活码 |

```python
ok, update_info = wx.activate("LIC-XXXXXXXX-XXXXXXXX")
if not ok:
    logger.error("激活失败：%s", update_info)
    return
```

**激活模型**：激活成功后，授权信息（含有效期）会写入本地
`C:\ProgramData\weixinauto\license.json`。之后 `is_activated()` 全程**纯本地校验**
（签名 + 机器指纹 + 有效期），不再联网。推荐启动时这样 gate，避免每次都打服务器：

```python
if not wx.is_activated():        # 纯本地，毫秒级
    ok, info = wx.activate(key)  # 只有本地无效时才联网激活一次
    if not ok:
        return
```

**按设备指纹查有效期**（本地文件丢失时确认、或运维查询，不需要激活码）：

```python
status = wx.query_license_status()
# {"ok": True, "found": True, "status": "valid",
#  "expire_at": "2027-07-21T10:20:25Z", "days_left": 311, "expired": False}
if status and status.get("ok"):
    print("剩余天数：", status["days_left"])
```
> `status` 取值：`valid` 有效 / `expired` 已过期 / `disabled` 秘钥禁用 /
> `activation_disabled` 本设备被禁用 / `not_found` 未绑定任何秘钥。网络异常返回 `None`。

---

## 4. 发送消息 / 文件

```python
# 文本
wx.send_message(target="工作群", text="你好", wait_seconds=1.6) -> bool

# 文件（图片、文档、压缩包等，路径可绝对或相对）
wx.send_file(target="工作群", file_path=r"E:\report.pdf", wait_seconds=1.6) -> bool
```

- `target`：群名或好友昵称。
- 返回 `True`/`False` 表示是否发送成功。
- 内部自动完成：前置微信 → 搜索并进入目标会话 → 填入/粘贴内容 → 发送。

---

## 5. 三种监听模式

监听共有三种方式，按场景选一种即可。**回调函数签名各不相同，注意区分。**

### 5.1 单会话监听（回调模式）—— `add_listen_chat`

适合精准盯一个群/人，来消息立即回调。

```python
import time
from weixinauto import Wechat, ChatMessage

def on_new(group: str, msg: ChatMessage, reply):
    # group: 会话标题
    # msg:   ChatMessage 对象（见第 6 节）
    # reply(content, mode="text"): 直接回复当前会话
    print(f"[{group}] {msg.sender}: {msg.text}")
    if msg.text == "1" and not msg.is_self:   # 防回声：排除自己发的
        reply("收到1")                         # 文本
        # reply(r"E:\a.jpg", mode="image")     # 图片
        # reply(r"E:\a.zip", mode="file")      # 文件

wx = Wechat()
wx.activate("...")

wx.add_listen_chat(
    title="工作群",
    on_new=on_new,
    interval_sec=0.18,      # 轮询间隔
    need_nickname=False,    # True 会右键点头像取发件人昵称（有前台干扰，见注意事项）
    auto_save_image=False,
    auto_save_file=False,
)

try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    wx.stop_listen("工作群")
```

> 回调里的 `reply` 签名是 `reply(content: str, mode: str = "text") -> bool`，`mode` 可取 `"text"`（默认）/`"file"`/`"image"`。

### 5.2 批量监听（轮询模式）—— `add_listens`

适合同时盯多个群，主循环里统一拉取。**这种模式不用回调，靠 `get_listen_messages` 主动取。**

```python
import time
from weixinauto import Wechat

wx = Wechat()
wx.activate("...")

wx.add_listens(
    titles=["工作群", "测试群A", "测试群B"],   # 也可传单个字符串
    need_nickname=True,
    poll_sec=0.15,
    auto_save_image=False,
    auto_save_file=False,
)
wx.start_listen()

try:
    while True:
        # key 是 ChatWnd 对象
        all_msgs = wx.get_listen_messages(clear=True)
        for chat_wnd, msgs in all_msgs.items():
            for msg in msgs:
                print(f"[{chat_wnd.title}] {msg.sender}: {msg.text}")
                if msg.text == "在吗" and not msg.is_self:
                    chat_wnd.reply("在的")          # 通过 ChatWnd 回复
        time.sleep(0.5)
except KeyboardInterrupt:
    wx.stop_listen()   # 不传 title = 停掉全部监听
```

如果不想处理 `ChatWnd` 对象，用 `get_listen_message`（少个 s），key 直接是群名字符串：

```python
all_msgs = wx.get_listen_message(clear=True)   # {"工作群": [ChatMessage, ...], ...}
```

### 5.3 全局未读监听 —— `get_new_message`

盯微信左侧会话列表的小红点，未打开/被折叠的会话来消息也能抓到。适合做全局默认接待机器人。

```python
import time
from typing import List
from weixinauto import Wechat, ChatMessage

def on_global_new(chatwnd, msgs: List[ChatMessage]):
    # 注意：回调签名是 (chatwnd, msgs)，chatwnd 在前
    for m in msgs:
        print(f"[{m.group}] {m.sender}: {m.text}")
        if m.text == "在吗" and not m.is_self:
            chatwnd.reply("您好，自动接待中")

wx = Wechat()
wx.activate("...")

wx.get_new_message(
    on_new=on_global_new,
    interval_sec=5.0,       # 扫描红点的间隔
    need_nickname=True,
    auto_save_image=False,
    auto_save_file=False,
)

try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    wx.stop_new_message_listen()
```

> ⚠️ 全局监听的回调是 `(chatwnd, msgs)`——**chatwnd 在前，消息列表在后**。（旧文档里写的 `(msgs, reply)` 已过时。）

#### 工作机理：它靠"小红点"驱动（理解这点才能用好）

全局监听不是推送，是**轮询左侧会话列表的未读红点**：

1. 每 `interval_sec` 扫一轮会话列表，找**第一个有未读的会话**
2. 点进去读取未读消息 → **收起面板** → 把消息交给你的回调
3. 免打扰会话直接跳过，不点击（不会被标记已读）

由此产生三条你必须知道的行为约定：

**① 每轮只处理一个会话。** 多个群同时有未读时，一轮消化一个。按默认
`interval_sec=5.0` 算，N 个群积压时最长要 N×5 秒才轮到最后一个。
急需更快就把 `interval_sec` 调小（但会更频繁地抢前台）。

**② 点进会话这个动作本身会消耗红点。** 面板打开期间到达的新消息会被微信
**立刻标记已读、不再产生红点**。SDK 会在收起面板前补捞一次，但这是尽力而为——
**极端情况下（对方在你回复的瞬间连发）仍可能漏**。面板一收起，新消息就会重新
产生红点，由下一轮稳稳捞到。

**③ 你自己发消息会打断它。** 如果你的程序用 `wx.send_message()` 发消息，
会和监听抢微信界面。SDK 内部有进程级 UI 锁保证不会同时操作，但会互相排队。

> **结论**：把它当成"**高概率、非100%**的消息通道"。业务上要能容忍偶发漏接
> （比如让对方重发、或用消息内容而非到达顺序做状态机），不要设计成"漏一条就出错"。

#### 投递顺序的约定

- 回调是**串行**的：上一批处理完（你的函数返回）才会投递下一批，
  **绝不会在你的回调还没返回时再次调用它**。所以你的回调不需要写成可重入。
- 你在回调里调用 `chatwnd.reply()` 之后，SDK 可能补捞到"回复期间新到的消息"。
  这些消息会作为**独立的下一批**投递，而不是塞进当前这批。
- 因此：**不要假设"一批 = 那一刻的全部消息"**。按消息本身处理，别依赖批次边界。

---

## 6. `ChatMessage` 消息对象

回调和拉取拿到的每条消息都是 `ChatMessage`：

| 属性 | 类型 | 说明 |
|---|---|---|
| `group` | `str` | 所属会话标题 |
| `text` | `str` | 解析后的纯文本 |
| `raw_text` | `str` | 原始文本（含换行） |
| `mtype` | `str` | 消息类型，见 `MessageType` |
| `is_self` | `bool\|None` | 是否自己发的（防回声用；`need_nickname=False` 时可能为 None） |
| `sender` | `str\|None` | 发件人昵称（仅 `need_nickname=True` 时有） |
| `ts` | `float` | 时间戳 |
| `file_name` | `str\|None` | 文件消息的文件名 |
| `file_path` | `str\|None` | 文件/图片保存后的本地路径 |

便捷判断（不必再写 `msg.mtype == MessageType.IMAGE` 这种字符串比较）：

```python
msg.is_text     # 是否文本
msg.is_image    # 是否图片
msg.is_file     # 是否文件
msg.is_voice    # 是否语音
msg.is_video    # 是否视频
msg.is_system   # 是否系统消息
msg.has_file    # 是否已有本地文件（file_path 非空）
msg.datetime    # ts 对应的 datetime 对象
```

方法：

```python
msg.save_image(base_dir=None) -> str|None   # 保存图片，返回本地路径
msg.save_file(base_dir=None)  -> str|None   # 保存文件，返回本地路径
msg.forward(to_contact, additional_message="") -> bool  # 转发给指定联系人
msg.to_dict() -> dict                       # 转可 JSON 序列化的 dict（落库 / 推队列用）
```

`to_dict()` 返回示例：

```python
{
    "group": "工作群", "sender": "张三", "text": "你好", "raw_text": "你好",
    "mtype": "text", "ts": 1700000000.0, "time": "2023-11-15 06:13:20",
    "is_self": False, "file_name": None, "file_path": None,
}
```

> 以下划线开头的属性（`_key` / `_node` / `_actions`）属于内部实现细节，业务侧请勿依赖。

---

## 7. `ChatWnd` 会话对象

批量监听 `get_listen_messages` 的 key、以及全局监听回调的 `chatwnd` 都是 `ChatWnd`：

| 成员 | 说明 |
|---|---|
| `.title` | 会话标题 |
| `.reply(text)` / `.reply_text(text)` | 回文本（两者等价） |
| `.reply_file(path)` | 回文件 |
| `.reply_image(path)` | 回图片 |

---

## 8. `MessageType` 消息类型常量

```python
from weixinauto import MessageType

MessageType.TEXT          # "text"     文本
MessageType.IMAGE         # "image"    图片
MessageType.VOICE         # "voice"    语音
MessageType.VIDEO         # "video"    视频
MessageType.FILE          # "file"     文件
MessageType.SYSTEM        # "system"   系统消息
MessageType.TIME          # "time"     时间分隔条
MessageType.TRANSFER      # "transfer" 转账
MessageType.RED_ENVELOPE  # "hongbao"  红包
```

`MessageType` 是 `StrEnum`，完全兼容原先当字符串用的写法，同时多了 IDE 补全和取值约束：

```python
msg.mtype == MessageType.IMAGE   # True
msg.mtype == "image"             # True（仍然成立）
str(msg.mtype)                   # "image"
json.dumps({"t": msg.mtype})     # {"t": "image"}
```

---

## 9. 方法速查表

| 方法 | 作用 | 返回 |
|---|---|---|
| `Wechat()` | 创建门面对象（driver 懒加载） | — |
| `activate(key, ...)` | 激活授权 | `(bool, dict\|None)` |
| `is_activated()` | 本地是否已激活（纯本地，不联网） | `bool` |
| `ensure_activated()` | 未激活抛异常 | `None` |
| `query_license_status(*, timeout=8.0)` | 按设备指纹联网查授权状态 | `dict\|None` |
| `send_message(target, text, *, wait_seconds=1.6)` | 发文本 | `bool` |
| `send_file(target, file_path, *, wait_seconds=1.6)` | 发文件 | `bool` |
| `open_or_focus_child_window(title)` | 打开/前置聊天子窗口 | `bool` |
| `close_chat_window(title)` | 关闭聊天子窗口 | `bool` |
| `add_listen_chat(title, on_new, *, interval_sec=0.18, need_nickname=False, auto_save_image=False, auto_save_file=False)` | 单会话回调监听 | `bool` |
| `add_listens(titles, *, need_nickname=True, poll_sec=0.15, auto_save_image=False, auto_save_file=False)` | 批量轮询监听 | `None` |
| `start_listen()` | 启动所有监听线程 | `None` |
| `get_listen_messages(clear=True)` | 取批量监听新消息（key=ChatWnd） | `Dict[ChatWnd, List[ChatMessage]]` |
| `get_listen_message(clear=True)` | 同上但 key=群名字符串 | `Dict[str, List[ChatMessage]]` |
| `stop_listen(title=None, *, join=True, timeout=1.5)` | 停监听（不传 title 停全部） | `None` |
| `get_new_message(on_new, *, interval_sec=1.0, need_nickname=False, auto_save_image=False, auto_save_file=False)` | 全局未读红点监听 | `bool` |
| `stop_new_message_listen()` | 停全局监听 | `None` |
| `shutdown()` | 停掉所有监听并清理驱动 | `None` |

---

## 10. 注意事项

1. **仅 Windows + 新版微信**，微信需保持登录、前台可见可用。
2. **必须先激活**，否则所有涉及微信操作的方法直接返回失败值（不抛异常）。
3. **防回声**：业务里回复前建议判断 `if not msg.is_self`，避免机器人把自己发的消息当成新指令死循环。
4. **`need_nickname=True` 的代价**：为取发件人昵称，会对头像做（虚拟）右键点击弹菜单读取，对前台有一定干扰、也更慢。不需要昵称就设 `False` 可无感挂机。
5. **不要把它当成 100% 可靠的消息通道**：本方案基于 UI 树物理抓取 + 未读红点轮询，
   极端场景（对方在回复瞬间连发、图片下载或取昵称占用前台期间）仍可能漏消息。
   机理与规避见 [5.3 的"工作机理"](#工作机理它靠小红点驱动理解这点才能用好)。
   业务层请做容错，别设计成"漏一条就出错"。
6. **日志**：见下面第 11 节。
7. **退出**：程序结束前调用 `wx.shutdown()` 清理监听线程与驱动。

---

## 11. 日志规范

作为通用组件，**本库导入时不会配置任何日志 handler，也不会创建日志目录**。
所有日志发往名为 `weixinauto` 的 logger，由你决定怎么输出。

### 方式一：用内置便捷配置（推荐给脚本 / 小型应用）

```python
import weixinauto

weixinauto.setup_logging(
    level="INFO",                    # 日志级别
    console=True,                    # 是否输出到控制台
    log_file="logs/weixinauto.log",  # 不传则不写文件；按天切割，默认保留 7 份
)
```

重复调用是安全的（幂等，不会重复挂 handler 导致日志翻倍）。

### 方式二：用标准 logging 自行接管（推荐给已有日志体系的项目）

```python
import logging

logging.basicConfig(level=logging.INFO)
logging.getLogger("weixinauto").setLevel(logging.DEBUG)   # 只调本库的级别
```

### ⚠️ 从 2.3.x 升级必读

2.3.x 及更早版本在 **import 的瞬间**就会在当前工作目录创建 `logs/` 并写
`logs/weixinauto.log`，还会强制把日志级别设为 INFO、关闭 propagate。
这对一个通用组件是不合适的（污染使用方目录、劫持使用方的日志配置），
**2.4.0 起已移除该行为**。

如果你原先依赖这个自动文件日志，在程序启动处加一行即可完全恢复：

```python
weixinauto.setup_logging(level="INFO", log_file="logs/weixinauto.log")
```

### 日志名约定

每个模块的 logger 名就是它的模块路径，便于按模块精细控制：

```
weixinauto.wechat
weixinauto.infra.uia.driver
weixinauto.infra.uia.chat_listener
weixinauto.infra.uia.subwindow_ops
```

例如只想看驱动层的 debug、其它保持 INFO：

```python
logging.getLogger("weixinauto").setLevel(logging.INFO)
logging.getLogger("weixinauto.infra.uia.driver").setLevel(logging.DEBUG)
```

---

## 12. 集成到其它项目的实践要点

### 12.1 一个可直接抄的骨架

```python
import logging
import weixinauto
from weixinauto import Wechat, ChatMessage

LICENSE_KEY = "LIC-XXXXXXXX-XXXXXXXX"   # 建议从配置/环境变量读，别硬编码进代码

# 1) 日志：库本身不配置日志，由你决定。接入已有体系就用标准 logging
weixinauto.setup_logging(level="INFO", log_file="logs/weixinauto.log")
logger = logging.getLogger(__name__)

wx = Wechat()

# 2) 激活：只在本地无效时才联网，避免每次启动都打授权服务器
if not wx.is_activated():
    ok, info = wx.activate(LICENSE_KEY)
    if not ok:
        logger.error("激活失败：%s", info)
        raise SystemExit(1)

# 3) 挂监听
def on_global(chatwnd, msgs):
    for m in msgs:
        if m.is_self:           # 防回声：跳过自己发的
            continue
        try:
            handle(m, chatwnd)  # 你的业务
        except Exception:
            logger.exception("处理消息失败: %s", m.text)   # 千万别让异常冒出去

wx.get_new_message(on_new=on_global, interval_sec=5.0, need_nickname=True)
```

### 12.2 六条踩过坑的建议

1. **回调里必须自己兜住异常**。回调抛出的异常只会被记一条 warning，
   但那一批消息就丢了。务必 `try/except` 包住业务逻辑。

2. **回调要快**。它是串行的——你的函数不返回，下一轮扫描就不会开始。
   调 LLM、查库这类耗时操作建议**丢进队列异步处理**，回调本身只负责接收：

   ```python
   def on_global(chatwnd, msgs):
       for m in msgs:
           if not m.is_self:
               task_queue.put(m.to_dict())   # 立刻返回，别在这里阻塞
   ```

3. **激活码别硬编码**。从配置文件或环境变量读；本地激活信息会存在
   `C:\ProgramData\weixinauto\license.json`，激活过一次后重启不需要再联网。

4. **`msg.to_dict()` 是跨进程/落库的好朋友**。它返回纯 JSON 可序列化的 dict，
   适合直接丢消息队列或写数据库，不用自己拆字段。

5. **退出时调 `wx.shutdown()`**，否则监听线程会留着。
   配合 `atexit` 或框架的 shutdown 钩子。

6. **别在多个线程里乱调 SDK**。内部虽有进程级 UI 锁保证不会同时操作界面，
   但并发调用只会互相排队、并不会更快，反而让时序更难排查。
   推荐：**监听在一个线程，业务处理走队列**。

### 12.3 部署检查清单

- [ ] Python 版本 **锁定 3.11**（64 位 Windows），其它版本装不上
- [ ] 微信版本是 **4.1.15.13**（微信 → 设置 → 关于），其它版本不保证可用
- [ ] `requirements.txt` 里写 `weixinautox4==2.8.12`（固定版本，别用浮动版本）
- [ ] 目标机器已用该设备的激活码激活过（一机一码）
- [ ] 微信已登录、保持前台可见（最小化/锁屏会影响 UI 自动化）
- [ ] 程序启动处调用了 `setup_logging()`，否则默认不产生任何日志
- [ ] 业务回调内部有 try/except，且不做耗时操作

---

## 联系我们

本 SDK 为**商业授权**产品，采用一机一码：每台机器需要一个独立的激活码，
激活后授权信息保存在本机 `C:\ProgramData\weixinauto\license.json`，重启不需要重新联网。

### 价格与售后

| 项目 | 说明 |
|---|---|
| 价格 | **300 元/年**，一机一码 |
| 支持的微信版本 | **仅限微信 PC 版 4.1.15.13**，其它版本不保证可用 |
| 退款 | **购买后不支持退款** |

> ⚠️ 购买前请先确认微信版本：微信 → 设置 → 关于。
> 微信升级到其它版本后可能无法使用，请不要随意升级微信。

**获取激活码、续期、技术支持、定制开发**，通过以下任一方式联系：

| 方式 | |
|---|---|
| 邮箱 | zhilongfeng66@gmail.com |
| 微信 | agentaibot |
| PyPI | https://pypi.org/project/weixinautox4/ |
| GitHub | https://github.com/zhilong9966/weixinauto |

### 反馈问题时请带上这些

排查基本都靠日志，带齐下面几样能少来回好几轮：

1. **版本号**：`python -c "import weixinauto; print(weixinauto.__version__)"`
2. **日志文件**。先按 [第 11 节](#11-日志规范) 打开 DEBUG 日志：

   ```python
   import weixinauto
   weixinauto.setup_logging(level="DEBUG", log_file="logs/weixinauto.log")
   ```

   失败时日志里会有一行 `失败原因=...`，那一行最关键。
3. **微信版本**（微信 → 设置 → 关于）和 **Windows 版本**
4. **能不能读到消息**——读不到八成是无障碍树没建立，先看 [第 0 节](#0-开始之前先让微信可被读取)

> 日志里可能含有聊天内容和授权码，发出来之前请自己过一眼、按需脱敏。

---

## 许可

专有软件（Proprietary）。未经授权不得复制、分发、反编译或用于再分发。
购买的授权仅限于在**已激活的机器上**按约定用途使用。
