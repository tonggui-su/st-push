# st_notify — 通知组件（铃铛 + 弹窗，v3.0）

> 依赖 `st_push` 的推送通道和模块注册系统。方案设计见 `prd_st_push_2.md`。

## 简介

独立的站内通知组件。接收应用层注入的 `PushClient`（公共基础设施），
注册 "notify" 模块，在页面右上角注入铃铛图标，未读数角标，
点击打开弹窗，逐条点击已读。

通知是单向的（系统→用户），无需发送同步。`send_notification()` 经
UserChannelManage 解析 `to_user→[channel_id]`，投递到接收者的所有 channel。

## 用法

```python
from st_push import PushClient
from st_push.comp.notify import NotificationClient

# 应用层创建公共 PushClient（session 单例，一个会话一个 WS 连接）
push_client = PushClient.session_instance(user="alice")

# 注入给通知组件
notifier = NotificationClient(push_client, user_name="alice")

# 渲染铃铛图标，注册 on_click 回调
def handle_click(payload):
    st.toast(f"点击了通知按钮: {payload}")
    # 例如：st.query_params["task_id"] = payload["task_id"]

notifier.render(on_click=handle_click)

# 发送通知给指定用户（经 UserChannelManage 解析 to_user→channels）
notifier.send_notification(
    to_user="bob",
    title="新任务分配",
    body="您被分配了「修复登录Bug」任务",
    level="info",
    action="查看任务",
    payload={"task_id": 123, "url": "/task/123"},
)

# 广播给所有在线用户
notifier.broadcast_alert(
    title="系统维护",
    body="服务器将于 10 分钟后重启",
    level="warning",
)
```

## 运行演示

```bash
python demos/notification_app.py
```

## API

| 方法 | 说明 |
|------|------|
| `NotificationClient(push_client, user_name, key="st_notify")` | 创建通知客户端，注入公共 PushClient，自动注册 "notify" 模块 |
| `render(on_click=None) -> bool` | 渲染铃铛图标，注册按钮点击回调，返回本次 rerun 是否有点击 |
| `send_notification(to_user, title, body="", level="info", action=None, payload=None, ...)` | 发送通知给指定用户（经 UserChannelManage 解析 to_user→channels） |
| `broadcast_alert(title, body="", level="warning", ...)` | 广播告警给所有在线 channel |
| `is_online(user) -> bool` | 查询用户是否有在线 channel |
| `module_id -> str` | 只读属性，模块 ID |
| `channel_id -> str` | 只读属性，复用 PushClient 的 channel ID |

### NotificationPayload

| 字段 | 说明 |
|------|------|
| `title` | 标题 |
| `body` | 内容 |
| `level` | 级别：info / success / warning / error |
| `action` | 操作按钮文字（可选，为空则不显示按钮） |
| `payload` | 按钮点击时传给 on_click 回调的数据（可选，任意 JSON 可序列化值） |
| `ts` | 时间戳 |

### on_click 回调

- 每条通知最多 1 个按钮（由 `action` 字段控制是否显示）
- 按钮点击后：前端 `setComponentValue` 触发 rerun → Python 侧调用 `on_click(payload)` → 返回 `True`
- 通过 `notificationId` 在 session_state 去重，避免同一点击在多次 rerun 中重复触发回调

## 交互

- 铃铛图标注入到 parent DOM（右上角）
- 未读数显示为角标
- 点击铃铛打开弹窗
- 每条通知必须点击才能标记已读
- 带按钮的通知，点击按钮触发 `on_click` 回调（点击按钮不会标记已读，需单独点击通知条目）

## 架构

- `st_push` 传输层：WebSocket 传输 + 模块路由（channel-based），PushCore 不知 user
- `st_push` 业务层：UserChannelManage 解析 `user→[channel_id]`，仅服务 PushClient
- `st_push` 接入层：PushClient（session 单例）+ Module，由应用层创建注入
- `st_notify` 接收注入的 PushClient，注册 "notify" 模块，渲染铃铛 + 弹窗 UI
- 消息通过 `st_push` 前端分发系统到达通知 callback
- 前端注入 parent DOM，通过 `setComponentValue` 回传用户操作

## 文件

| 文件 | 说明 |
|------|------|
| `__init__.py` | NotificationClient + NotificationPayload |
| `frontend/build/index.html` | 铃铛图标 + 弹窗 UI |
