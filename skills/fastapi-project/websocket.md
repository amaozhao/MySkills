---
name: fastapi-websocket
description: FastAPI WebSocket 最佳实践 - 连接管理、消息广播、实时通信
---

# FastAPI WebSocket

## 概述

WebSocket 是一种在单个 TCP 连接上进行全双工通信的协议，适用于实时聊天、通知推送、实时更新等场景。

## 基础用法

### 简单 WebSocket

```python
# app/api/websocket.py
from fastapi import APIRouter, WebSocket, WebSocketDisconnect

router = APIRouter()


@router.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()

    try:
        while True:
            # 接收消息
            data = await websocket.receive_text()

            # 发送消息
            await websocket.send_text(f"Echo: {data}")

    except WebSocketDisconnect:
        print("Client disconnected")
```

### 连接与断开

```python
# app/api/websocket.py
from fastapi import APIRouter, WebSocket, WebSocketDisconnect
from typing import List

router = APIRouter()


@router.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    # 1. 接受连接
    await websocket.accept()

    try:
        # 2. 循环接收消息
        while True:
            # 接收文本
            data = await websocket.receive_text()

            # 或接收 JSON
            # data = await websocket.receive_json()

            print(f"Received: {data}")

            # 发送响应
            await websocket.send_json({"status": "ok", "data": data})

    except WebSocketDisconnect:
        # 3. 客户端断开
        print(f"Client disconnected")
    except Exception as e:
        # 4. 发生错误
        await websocket.close(code=1011, reason=str(e))
```

---

## 连接管理器

### 管理多个连接

```python
# app/ws/manager.py
from fastapi import WebSocket
from typing import List, Dict, Set
import asyncio


class ConnectionManager:
    """WebSocket 连接管理器"""

    def __init__(self):
        # 活跃连接
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        """添加新连接"""
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        """移除连接"""
        if websocket in self.active_connections:
            self.active_connections.remove(websocket)

    async def send_personal_message(self, message: str, websocket: WebSocket):
        """发送个人消息"""
        await websocket.send_text(message)

    async def broadcast(self, message: str):
        """广播消息给所有连接"""
        for connection in self.active_connections:
            try:
                await connection.send_text(message)
            except Exception:
                # 移除无效连接
                self.disconnect(connection)


# 全局管理器
manager = ConnectionManager()


# 使用
@router.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"Broadcast: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

### 按用户管理连接

```python
# app/ws/user_manager.py
from fastapi import WebSocket
from typing import Dict, List
import asyncio


class UserConnectionManager:
    """按用户 ID 管理连接"""

    def __init__(self):
        # user_id -> List[WebSocket]
        self.user_connections: Dict[int, List[WebSocket]] = {}

    async def connect(self, user_id: int, websocket: WebSocket):
        """用户连接"""
        await websocket.accept()

        if user_id not in self.user_connections:
            self.user_connections[user_id] = []

        self.user_connections[user_id].append(websocket)

    def disconnect(self, user_id: int, websocket: WebSocket):
        """用户断开"""
        if user_id in self.user_connections:
            if websocket in self.user_connections[user_id]:
                self.user_connections[user_id].remove(websocket)

            if not self.user_connections[user_id]:
                del self.user_connections[user_id]

    async def send_to_user(self, user_id: int, message: str):
        """发送消息给指定用户"""
        if user_id in self.user_connections:
            for connection in self.user_connections[user_id]:
                try:
                    await connection.send_text(message)
                except Exception:
                    self.disconnect(user_id, connection)

    async def broadcast(self, message: str):
        """广播给所有在线用户"""
        for user_id, connections in self.user_connections.items():
            for connection in connections:
                try:
                    await connection.send_text(message)
                except Exception:
                    self.disconnect(user_id, connection)


user_manager = UserConnectionManager()
```

---

## 房间/群组功能

### 房间管理

```python
# app/ws/room_manager.py
from fastapi import WebSocket
from typing import Dict, Set
import asyncio


class RoomManager:
    """房间管理器 - 支持群组/频道"""

    def __init__(self):
        # room_id -> Set[WebSocket]
        self.rooms: Dict[str, Set[WebSocket]] = {}
        # WebSocket -> room_id
        self.connection_rooms: Dict[WebSocket, str] = {}

    async def join_room(self, room_id: str, websocket: WebSocket):
        """加入房间"""
        await websocket.accept()

        if room_id not in self.rooms:
            self.rooms[room_id] = set()

        self.rooms[room_id].add(websocket)
        self.connection_rooms[websocket] = room_id

        # 通知用户加入
        await self.broadcast_to_room(
            room_id,
            f"User joined room {room_id}"
        )

    def leave_room(self, websocket: WebSocket):
        """离开房间"""
        room_id = self.connection_rooms.get(websocket)
        if not room_id:
            return

        if room_id in self.rooms:
            self.rooms[room_id].discard(websocket)

            if not self.rooms[room_id]:
                del self.rooms[room_id]

        del self.connection_rooms[websocket]

    async def send_to_room(self, room_id: str, message: str, exclude: WebSocket = None):
        """发送消息到房间"""
        if room_id not in self.rooms:
            return

        for connection in self.rooms[room_id]:
            if connection == exclude:
                continue
            try:
                await connection.send_text(message)
            except Exception:
                self.leave_room(connection)

    async def broadcast_to_room(self, room_id: str, message: str):
        """广播到房间（包括发送者）"""
        if room_id in self.rooms:
            for connection in list(self.rooms[room_id]):
                try:
                    await connection.send_text(message)
                except Exception:
                    self.leave_room(connection)


room_manager = RoomManager()


# 使用
@router.websocket("/ws/room/{room_id}")
async def websocket_room(websocket: WebSocket, room_id: str):
    await room_manager.join_room(room_id, websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await room_manager.send_to_room(room_id, f"Room {room_id}: {data}")
    except WebSocketDisconnect:
        room_manager.leave_room(websocket)
```

---

## 认证与安全

### WebSocket 认证

```python
# app/ws/auth.py
from fastapi import WebSocket, HTTPException, status
from typing import Optional


async def get_websocket_current_user(websocket: WebSocket) -> User:
    """WebSocket 认证依赖"""
    # 方式1：从 Query 参数获取 Token
    token = websocket.query_params.get("token")
    if not token:
        await websocket.close(code=1008, reason="Missing token")
        raise HTTPException(status_code=401, detail="Missing token")

    # 验证 Token
    user = await verify_token(token)
    if not user:
        await websocket.close(code=1008, reason="Invalid token")
        raise HTTPException(status_code=401, detail="Invalid token")

    return user


# 使用
@router.websocket("/ws/auth")
async def websocket_auth(websocket: WebSocket):
    # 认证
    user = await get_websocket_current_user(websocket)

    await websocket.accept()
    # ... 处理连接
```

### 认证后的连接管理

```python
# app/ws/authenticated_manager.py
from fastapi import WebSocket
from app.ws.user_manager import UserConnectionManager
from app.core.security import verify_token

user_manager = UserConnectionManager()


@router.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    # 从 Query 获取 token
    token = websocket.query_params.get("token")
    if not token:
        await websocket.close(code=1008)
        return

    # 验证并获取用户
    user = await verify_token(token)
    if not user:
        await websocket.close(code=1008)
        return

    # 连接
    await user_manager.connect(user.id, websocket)

    try:
        while True:
            data = await websocket.receive_text()
            # 处理消息
            await handle_message(user.id, data)
    except Exception:
        pass
    finally:
        user_manager.disconnect(user.id, websocket)
```

---

## 消息处理

### JSON 消息格式

```python
# app/ws/messages.py
from pydantic import BaseModel
from typing import Optional, Literal
from enum import Enum


class MessageType(str, Enum):
    CHAT = "chat"
    NOTIFICATION = "notification"
    TYPING = "typing"
    PRESENCE = "presence"


class WSMessage(BaseModel):
    """WebSocket 消息格式"""
    type: MessageType
    payload: dict
    sender_id: Optional[int] = None


@router.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()

    try:
        while True:
            # 接收 JSON 消息
            data = await websocket.receive_json()

            # 解析消息
            message = WSMessage(**data)

            # 处理不同类型消息
            if message.type == MessageType.CHAT:
                await handle_chat(message)
            elif message.type == MessageType.TYPING:
                await handle_typing(message)

    except Exception as e:
        await websocket.send_json({
            "type": "error",
            "payload": {"message": str(e)}
        })
```

### 心跳保活

```python
# app/ws/heartbeat.py
import asyncio
from fastapi import WebSocket, WebSocketDisconnect


class HeartbeatWebSocket:
    """带心跳的 WebSocket"""

    def __init__(self, websocket: WebSocket, ping_interval: int = 30):
        self.websocket = websocket
        self.ping_interval = ping_interval
        self.last_pong = asyncio.get_event_loop().time()

    async def listen(self):
        """监听消息，带心跳检测"""

        async def ping():
            """定期发送 ping"""
            while True:
                await asyncio.sleep(self.ping_interval)
                try:
                    await self.websocket.send_json({"type": "ping"})
                    self.last_pong = asyncio.get_event_loop().time()
                except Exception:
                    break

        # 启动 ping 任务
        ping_task = asyncio.create_task(ping())

        try:
            while True:
                data = await self.websocket.receive_text()
                # 处理消息

        except WebSocketDisconnect:
            ping_task.cancel()
        except Exception:
            ping_task.cancel()
            raise


# 客户端需要响应 pong
# setInterval(() => ws.send(JSON.stringify({type: 'pong'})), 30000)
```

---

## 实际应用示例

### 实时聊天

```python
# app/api/chat.py
from fastapi import APIRouter, WebSocket, Depends
from pydantic import BaseModel
from typing import Optional

router = APIRouter()


class ChatMessage(BaseModel):
    room_id: str
    content: str
    sender_id: int
    sender_name: str


class ChatManager:
    """聊天管理器"""

    def __init__(self):
        self.rooms: dict[str, set[WebSocket]] = {}

    async def join(self, room_id: str, ws: WebSocket):
        await ws.accept()
        if room_id not in self.rooms:
            self.rooms[room_id] = set()
        self.rooms[room_id].add(ws)

    async def send_message(self, room_id: str, message: ChatMessage):
        if room_id in self.rooms:
            for ws in self.rooms[room_id]:
                await ws.send_json({
                    "type": "message",
                    "data": message.model_dump()
                })


chat_manager = ChatManager()


@router.websocket("/chat/{room_id}")
async def chat_websocket(websocket: WebSocket, room_id: str):
    await chat_manager.join(room_id, websocket)
    try:
        while True:
            data = await websocket.receive_json()
            message = ChatMessage(**data, room_id=room_id)
            await chat_manager.send_message(room_id, message)
    except Exception:
        pass
```

### 实时通知

```python
# app/ws/notifications.py
from fastapi import WebSocket
from app.ws.user_manager import user_manager
import asyncio


async def send_notification(user_id: int, notification: dict):
    """发送通知给用户"""
    await user_manager.send_to_user(
        user_id,
        {
            "type": "notification",
            "payload": notification
        }
    )


async def broadcast_notification(notification: dict):
    """广播通知给所有用户"""
    await user_manager.broadcast({
        "type": "notification",
        "payload": notification
    })


# 使用：在业务逻辑中调用
# await send_notification(user_id, {"title": "New message", "body": "..."})
```

---

## 错误处理

### 连接错误处理

```python
# app/ws/errors.py
from fastapi import WebSocket, WebSocketDisconnect
import logging

logger = logging.getLogger(__name__)


@router.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    try:
        await websocket.accept()

        while True:
            data = await websocket.receive_text()
            # 处理消息

    except WebSocketDisconnect:
        logger.info(f"Client disconnected normally")

    except Exception as e:
        logger.error(f"WebSocket error: {e}")
        # 尝试关闭连接
        try:
            await websocket.close(code=1011, reason="Internal error")
        except Exception:
            pass
```

### 重连机制

```python
# 前端 JavaScript 示例
class WebSocketClient {
    constructor(url) {
        this.url = url;
        this.ws = null;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 5;
    }

    connect() {
        this.ws = new WebSocket(this.url);

        this.ws.onclose = () => {
            if (this.reconnectAttempts < this.maxReconnectAttempts) {
                this.reconnectAttempts++;
                setTimeout(() => this.connect(), 1000 * this.reconnectAttempts);
            }
        };

        this.ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            this.handleMessage(data);
        };
    }
}
```

---

## 性能优化

### 连接限制

```python
# app/ws/limits.py
from fastapi import WebSocket, HTTPException

MAX_CONNECTIONS = 1000

connection_count = 0


@router.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    global connection_count

    if connection_count >= MAX_CONNECTIONS:
        await websocket.close(code=1013, reason="Server full")
        raise HTTPException(status_code=503, detail="Server at capacity")

    connection_count += 1
    try:
        # 处理连接
        ...
    finally:
        connection_count -= 1
```

### 消息队列

```python
# app/ws/queue.py
import asyncio
from fastapi import WebSocket
from collections import deque


class MessageQueue:
    """消息队列，削峰填谷"""

    def __init__(self, max_size: int = 100):
        self.queue = deque(maxlen=max_size)
        self.processing = False

    async def enqueue(self, message: dict):
        self.queue.append(message)

        if not self.processing:
            asyncio.create_task(self.process())

    async def process(self):
        self.processing = True

        while self.queue:
            message = self.queue.popleft()
            await self.handle_message(message)
            await asyncio.sleep(0.1)  # 限速

        self.processing = False
```

---

## 最佳实践

### 1. 保持连接轻量

```python
# ❌ 避免：在连接中做重操作
@router.websocket("/ws")
async def bad_websocket(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        result = await heavy_database_query()  # 不要在这里做重操作
        await websocket.send_json(result)
```

### 2. 异常处理

```python
# ✅ 正确：做好异常处理和清理
@router.websocket("/ws")
async def good_websocket(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            # 业务逻辑
    except WebSocketDisconnect:
        pass  # 正常断开
    except Exception:
        logger.exception("WebSocket error")
    finally:
        # 清理资源
        cleanup(websocket)
```

### 3. 监控连接状态

```python
# 记录连接数指标
from prometheus_client import Counter, Gauge

ws_connections = Gauge('ws_connections', 'Current WebSocket connections')

@router.websocket("/ws")
async def monitored_websocket(websocket: WebSocket):
    ws_connections.inc()
    try:
        ...
    finally:
        ws_connections.dec()
```

---

## 相关文件

- [auth.md](./auth.md) - 认证实现
- [DI.md](./DI.md) - 依赖注入
- [middleware.md](./middleware.md) - 中间件
- [config.md](./config.md) - 配置管理
