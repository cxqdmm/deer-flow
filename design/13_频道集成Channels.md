# IM 频道集成（Channels）

---

## 1. 概述

DeerFlow 支持通过即时通讯应用与用户交互。目前支持四个平台：

| 平台 | 传输方式 | 上手难度 | Streaming |
|------|---------|---------|-----------|
| **Telegram** | Bot API（Long Polling） | 简单 | ❌ |
| **Slack** | Socket Mode | 中等 | ❌ |
| **飞书/Lark** | WebSocket | 中等 | ✅ |
| **企业微信** | WebSocket | 中等 | ✅ |

频道服务在 Gateway 启动时自动初始化，无需公网回调地址。

---

## 2. 架构

```
IM 平台
    │
    ├── Telegram ──── Long Polling ──────┐
    ├── Slack ───────── Socket Mode ────┼──→ ChannelManager
    ├── 飞书 ────────── WebSocket ──────┤    │
    └── 企业微信 ────── WebSocket ──────┘    │
                                            ▼
                                     MessageBus
                                     （消息总线）
                                            │
                              ┌─────────────┼─────────────┐
                              ▼             ▼             ▼
                        LangGraph    LangGraph    LangGraph
                        Server       Server       Server
                              │             │             │
                              └─────────────┼─────────────┘
                                            ▼
                                     Agent 响应
                                            │
                              ┌─────────────┼─────────────┐
                              ▼             ▼             ▼
                           Telegram       Slack         飞书
```

---

## 3. MessageBus（消息总线）

```python
# app/channels/message_bus.py

class MessageBus:
    """异步消息总线"""

    # 入站消息队列
    inbound: asyncio.Queue[InboundMessage]

    # 出站消息回调
    outbound_callbacks: list[Callable]

    async def publish_inbound(self, message: InboundMessage):
        """接收 IM 平台消息，加入处理队列"""

    async def publish_outbound(self, message: OutboundMessage):
        """发送响应到 IM 平台"""
        for callback in outbound_callbacks:
            await callback(message)
```

### 3.1 消息类型

```python
@dataclass
class InboundMessage:
    channel: str           # "telegram" / "slack" / "feishu" / "wecom"
    chat_id: str           # 聊天 ID
    user_id: str            # 用户 ID
    text: str               # 文本内容
    attachments: list        # 文件/图片附件
    message_id: str        # 平台消息 ID

@dataclass
class OutboundMessage:
    channel: str
    chat_id: str
    text: str
    is_final: bool         # 最终响应？流式更新？
    message_id: str | None  # 用于更新已有消息
```

---

## 4. ChannelManager（频道管理器）

```python
# app/channels/manager.py

class ChannelManager:
    """消费 MessageBus 队列，调度到 LangGraph Server"""

    async def _dispatch_loop():
        """主循环：消费入站消息"""
        while True:
            message = await message_bus.inbound.get()
            await _handle_message(message)
```

### 4.1 消息处理流程

```python
async def _handle_message(message: InboundMessage):
    # 1. 解析命令（/new, /status, /models, /memory, /help）
    if is_command(message):
        return await _handle_command(message)

    # 2. 查找/创建 LangGraph thread
    thread_id = store.get_or_create_thread(message)

    # 3. 发送到 LangGraph
    if supports_streaming(message.channel):
        # 飞书/企业微信：流式
        await _stream_to_channel(thread_id, message)
    else:
        # Slack/Telegram：等待完成
        await _wait_and_reply(thread_id, message)
```

---

## 5. 各平台实现

### 5.1 Telegram

```python
# app/channels/telegram.py

class TelegramChannel(BaseChannel):
    async def start(self):
        """Long Polling 循环"""
        offset = 0
        while True:
            updates = await self.bot.get_updates(offset=offset)
            for update in updates:
                await self._handle_update(update)
                offset = update.update_id + 1
            await asyncio.sleep(1)
```

特点：
- 无需公网地址
- Long Polling 方式获取更新
- 不支持流式（等待 Agent 完成再回复）

### 5.2 Slack

```python
# app/channels/slack.py

class SlackChannel(BaseChannel):
    async def start(self):
        """Socket Mode"""
        async with self.client.websocket as ws:
            await ws.connect()
            async for message in ws:
                await self._handle_event(json.loads(message))
```

特点：
- Socket Mode（不需要公网地址）
- App-Level Token 管理
- 支持文件接收

### 5.3 飞书/Lark

```python
# app/channels/feishu.py

class FeishuChannel(BaseChannel):
    async def _on_message(self, message: dict):
        """WebSocket 接收消息"""
        # 流式更新：同一卡片持续 patch
        await self._send_running_card(message_id)
        # ...
        await self._patch_final_card(message_id, final_response)
```

特点：
- WebSocket 长连接
- **支持流式**：同一消息卡片持续 patch 更新
- `message_id` 跟踪机制保证更新到同一卡片

### 5.4 企业微信

```python
# app/channels/wecom.py

class WeComChannel(BaseChannel):
    """企业微信智能机器人"""
```

特点：
- WebSocket 长连接
- 支持文本、图片、文件入站
- Agent 生成的图片/文件自动回传

---

## 6. Thread 持久化

```python
# app/channels/store.py

class ChannelStore:
    """JSON 文件持久化：channel:chat_id → thread_id"""

    def get_or_create_thread(message: InboundMessage) -> str:
        """查找或创建 LangGraph thread"""
        key = f"{message.channel}:{message.chat_id}"
        if key in self._store:
            return self._store[key]
        # 创建新 thread
        thread = await langgraph_client.threads.create()
        self._store[key] = thread.thread_id
        return thread.thread_id
```

---

## 7. 配置

### 7.1 config.yaml

```yaml
channels:
  langgraph_url: http://localhost:2024
  gateway_url: http://localhost:8001

  # 全局 session 默认值
  session:
    assistant_id: lead_agent
    config:
      recursion_limit: 100
    context:
      thinking_enabled: true
      is_plan_mode: false
      subagent_enabled: false

  # 各平台配置...
  telegram:
    enabled: false
    bot_token: $TELEGRAM_BOT_TOKEN
    allowed_users: []

  slack:
    enabled: false
    bot_token: $SLACK_BOT_TOKEN
    app_token: $SLACK_APP_TOKEN

  feishu:
    enabled: false
    app_id: $FEISHU_APP_ID
    app_secret: $FEISHU_APP_SECRET

  wecom:
    enabled: false
    bot_id: $WECOM_BOT_ID
    bot_secret: $WECOM_BOT_SECRET
```

### 7.2 Per-User 配置

```yaml
channels:
  telegram:
    enabled: true
    session:
      users:
        "123456789":
          assistant_id: vip-agent
          config:
            recursion_limit: 150
          context:
            thinking_enabled: true
            subagent_enabled: true
```

---

## 8. 源码索引

| 文件 | 说明 |
|------|------|
| `channels/manager.py` | ChannelManager + 消息调度 |
| `channels/message_bus.py` | MessageBus（消息总线） |
| `channels/store.py` | ChannelStore（thread 持久化） |
| `channels/base.py` | BaseChannel 抽象类 |
| `channels/telegram.py` | Telegram 实现 |
| `channels/slack.py` | Slack 实现 |
| `channels/feishu.py` | 飞书/Lark 实现 |
| `channels/wecom.py` | 企业微信实现 |
| `channels/service.py` | 频道服务生命周期 |

---

*文档版本：v2.0 | 生成日期：2026/04/08*
