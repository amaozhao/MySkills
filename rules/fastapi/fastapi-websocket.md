---
name: fastapi-websocket
description: FastAPI WebSocket 最佳实践 - 连接管理、消息广播、实时通信
paths: **/*.py
---

# FastAPI WebSocket

## 概述

WebSocket 是一种在单个 TCP 连接上进行全双工通信的协议，适用于实时聊天、通知推送等场景。

## 基础用法

```python
# app/api/websocket.py
from fastapi import APIRouter, WebSocket, WebSocketDisconnect

router = APIRouter()


@router.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()

    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Echo: {data}")
    except WebSocketDisconnect:
        print("Client disconnected")
```

## 连接管理器

```python
# app/ws/manager.py
from typing import List, Dict
from fastapi import WebSocket


class ConnectionManager:
    """WebSocket 连接管理器"""

    def __init__(self):
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        if websocket in self.active_connections:
            self.active_connections.remove(websocket)

    async def broadcast(self, message: str):
        for connection in self.active_connections:
            try:
                await connection.send_text(message)
            except Exception:
                self.disconnect(connection)


manager = ConnectionManager()


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

## 按用户管理连接

```python
# app/ws/user_manager.py
from typing import Dict, List
from fastapi import WebSocket


class UserConnectionManager:
    """按用户 ID 管理连接"""

    def __init__(self):
        self.user_connections: Dict[int, List[WebSocket]] = {}

    async def connect(self, user_id: int, websocket: WebSocket):
        await websocket.accept()
        if user_id not in self.user_connections:
            self.user_connections[user_id] = []
        self.user_connections[user_id].append(websocket)

    def disconnect(self, user_id: int, websocket: WebSocket):
        if user_id in self.user_connections:
            if websocket in self.user_connections[user_id]:
                self.user_connections[user_id].remove(websocket)
            if not self.user_connections[user_id]:
                del self.user_connections[user_id]

    async def send_to_user(self, user_id: int, message: str):
        if user_id in self.user_connections:
            for connection in self.user_connections[user_id]:
                try:
                    await connection.send_text(message)
                except Exception:
                    self.disconnect(user_id, connection)
```

## WebSocket 认证

```python
# app/ws/auth.py
async def get_websocket_current_user(websocket: WebSocket) -> User:
    token = websocket.query_params.get("token")
    if not token:
        await websocket.close(code=1008, reason="Missing token")
        raise HTTPException(status_code=401)

    user = await verify_token(token)
    if not user:
        await websocket.close(code=1008, reason="Invalid token")
        raise HTTPException(status_code=401)
    return user


@router.websocket("/ws/auth")
async def websocket_auth(websocket: WebSocket):
    user = await get_websocket_current_user(websocket)
    await websocket.accept()
    # ... 处理连接
```

## 错误处理

```python
@router.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    try:
        await websocket.accept()
        while True:
            data = await websocket.receive_text()
            # 处理消息
    except WebSocketDisconnect:
        logger.info("Client disconnected normally")
    except Exception as e:
        logger.error(f"WebSocket error: {e}")
        try:
            await websocket.close(code=1011, reason="Internal error")
        except Exception:
            pass
```

## 最佳实践

1. **保持连接轻量**：不要在连接中做重操作
2. **做好异常处理**：确保资源清理
3. **心跳保活**：使用 ping/pong 机制
4. **连接限制**：防止恶意占用资源
