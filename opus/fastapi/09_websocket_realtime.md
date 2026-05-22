# FastAPI WebSocket & Real-Time — Senior Level

## 1. WebSocket Basics

### Simple WebSocket Endpoint
```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
import json

app = FastAPI()


@app.websocket("/ws/{user_id}")
async def websocket_endpoint(websocket: WebSocket, user_id: str):
    await websocket.accept()
    
    try:
        while True:
            # Receive message from client
            data = await websocket.receive_text()
            message = json.loads(data)
            
            # Echo back (or process)
            await websocket.send_json({
                "type": "echo",
                "user_id": user_id,
                "content": message,
            })
    except WebSocketDisconnect:
        print(f"Client {user_id} disconnected")
```

### Connection Manager (Multi-Client)
```python
from fastapi import WebSocket
from typing import Dict, Set
import json


class ConnectionManager:
    """Manages WebSocket connections for real-time features."""
    
    def __init__(self):
        # user_id → set of WebSocket connections (multiple tabs/devices)
        self.active_connections: Dict[str, Set[WebSocket]] = {}
        # room_id → set of user_ids
        self.rooms: Dict[str, Set[str]] = {}
    
    async def connect(self, websocket: WebSocket, user_id: str):
        await websocket.accept()
        if user_id not in self.active_connections:
            self.active_connections[user_id] = set()
        self.active_connections[user_id].add(websocket)
    
    def disconnect(self, websocket: WebSocket, user_id: str):
        self.active_connections[user_id].discard(websocket)
        if not self.active_connections[user_id]:
            del self.active_connections[user_id]
            # Remove from all rooms
            for room_id, members in self.rooms.items():
                members.discard(user_id)
    
    async def send_to_user(self, user_id: str, message: dict):
        """Send message to all connections of a user."""
        connections = self.active_connections.get(user_id, set())
        disconnected = set()
        for ws in connections:
            try:
                await ws.send_json(message)
            except Exception:
                disconnected.add(ws)
        # Clean up broken connections
        for ws in disconnected:
            self.active_connections[user_id].discard(ws)
    
    async def broadcast_to_room(self, room_id: str, message: dict, exclude: str = None):
        """Send message to all users in a room."""
        members = self.rooms.get(room_id, set())
        for user_id in members:
            if user_id != exclude:
                await self.send_to_user(user_id, message)
    
    async def broadcast_all(self, message: dict):
        """Send to all connected users."""
        for user_id in self.active_connections:
            await self.send_to_user(user_id, message)
    
    def join_room(self, user_id: str, room_id: str):
        if room_id not in self.rooms:
            self.rooms[room_id] = set()
        self.rooms[room_id].add(user_id)
    
    def leave_room(self, user_id: str, room_id: str):
        if room_id in self.rooms:
            self.rooms[room_id].discard(user_id)
    
    def get_online_users(self) -> list[str]:
        return list(self.active_connections.keys())


manager = ConnectionManager()
```

---

## 2. Chat Application (Complete)

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Depends, Query
from datetime import datetime
import json

app = FastAPI()
manager = ConnectionManager()


@app.websocket("/ws/chat/{room_id}")
async def chat_websocket(
    websocket: WebSocket,
    room_id: str,
    token: str = Query(...),
):
    # Authenticate via query param (WebSocket can't use headers easily)
    try:
        user = await authenticate_ws_token(token)
    except Exception:
        await websocket.close(code=4001, reason="Unauthorized")
        return
    
    user_id = str(user.id)
    await manager.connect(websocket, user_id)
    manager.join_room(user_id, room_id)
    
    # Notify room that user joined
    await manager.broadcast_to_room(room_id, {
        "type": "user_joined",
        "user_id": user_id,
        "username": user.full_name,
        "timestamp": datetime.utcnow().isoformat(),
    }, exclude=user_id)
    
    try:
        while True:
            data = await websocket.receive_text()
            message = json.loads(data)
            
            if message["type"] == "chat_message":
                # Save to database
                saved = await save_message(room_id, user_id, message["content"])
                
                # Broadcast to room
                await manager.broadcast_to_room(room_id, {
                    "type": "chat_message",
                    "id": str(saved.id),
                    "user_id": user_id,
                    "username": user.full_name,
                    "content": message["content"],
                    "timestamp": saved.created_at.isoformat(),
                })
            
            elif message["type"] == "typing":
                await manager.broadcast_to_room(room_id, {
                    "type": "typing",
                    "user_id": user_id,
                    "username": user.full_name,
                }, exclude=user_id)
    
    except WebSocketDisconnect:
        manager.disconnect(websocket, user_id)
        manager.leave_room(user_id, room_id)
        
        await manager.broadcast_to_room(room_id, {
            "type": "user_left",
            "user_id": user_id,
            "username": user.full_name,
            "timestamp": datetime.utcnow().isoformat(),
        })
```

---

## 3. Scaled WebSocket with Redis Pub/Sub

### Problem: Multiple Server Instances
```
Without Redis:
  User A (Server 1) sends message
  User B (Server 2) never receives it
  (ConnectionManager is in-memory, per-process)

With Redis Pub/Sub:
  User A (Server 1) → publishes to Redis channel
  Redis → broadcasts to ALL servers subscribed
  Server 2 → delivers to User B
```

### Implementation
```python
import aioredis
import asyncio
import json
from contextlib import asynccontextmanager


class RedisWebSocketManager:
    """WebSocket manager backed by Redis for multi-server deployment."""
    
    def __init__(self, redis_url: str):
        self.redis_url = redis_url
        self.local_connections: Dict[str, Set[WebSocket]] = {}
        self.pubsub = None
        self._listener_task = None
    
    async def start(self):
        """Start Redis pub/sub listener."""
        self.redis = await aioredis.from_url(self.redis_url)
        self.pubsub = self.redis.pubsub()
        self._listener_task = asyncio.create_task(self._listen())
    
    async def stop(self):
        if self._listener_task:
            self._listener_task.cancel()
        if self.pubsub:
            await self.pubsub.close()
        await self.redis.close()
    
    async def _listen(self):
        """Background task: receive messages from Redis and deliver locally."""
        while True:
            try:
                message = await self.pubsub.get_message(
                    ignore_subscribe_messages=True, timeout=1.0
                )
                if message and message["type"] == "message":
                    data = json.loads(message["data"])
                    channel = message["channel"].decode()
                    
                    # Deliver to local connections
                    room_id = channel.replace("room:", "")
                    await self._deliver_local(room_id, data)
            except asyncio.CancelledError:
                break
            except Exception:
                await asyncio.sleep(1)
    
    async def _deliver_local(self, room_id: str, message: dict):
        """Deliver message to WebSocket connections on THIS server."""
        target_users = self.rooms.get(room_id, set())
        for user_id in target_users:
            connections = self.local_connections.get(user_id, set())
            for ws in connections:
                try:
                    await ws.send_json(message)
                except Exception:
                    pass
    
    async def connect(self, websocket: WebSocket, user_id: str):
        await websocket.accept()
        if user_id not in self.local_connections:
            self.local_connections[user_id] = set()
        self.local_connections[user_id].add(websocket)
    
    async def join_room(self, user_id: str, room_id: str):
        # Subscribe to Redis channel for this room
        await self.pubsub.subscribe(f"room:{room_id}")
        if not hasattr(self, 'rooms'):
            self.rooms = {}
        if room_id not in self.rooms:
            self.rooms[room_id] = set()
        self.rooms[room_id].add(user_id)
    
    async def publish_to_room(self, room_id: str, message: dict):
        """Publish message to Redis → ALL servers deliver to their local clients."""
        await self.redis.publish(f"room:{room_id}", json.dumps(message))
    
    async def publish_to_user(self, user_id: str, message: dict):
        """Direct message via Redis user channel."""
        await self.redis.publish(f"user:{user_id}", json.dumps(message))


# Lifespan
@asynccontextmanager
async def lifespan(app: FastAPI):
    ws_manager = RedisWebSocketManager("redis://redis:6379")
    await ws_manager.start()
    app.state.ws_manager = ws_manager
    yield
    await ws_manager.stop()
```

---

## 4. Server-Sent Events (SSE) — Simpler Alternative

### When to Use SSE vs WebSocket
```
SSE (Server-Sent Events):
  + Server → Client only (one-way)
  + Works over regular HTTP (no upgrade)
  + Auto-reconnect built into browser
  + Works through proxies/load balancers easily
  + Simpler implementation
  - Client can't send messages back (use REST for that)
  
  Best for: notifications, live feeds, dashboards, progress updates

WebSocket:
  + Bidirectional (client ↔ server)
  + Lower latency (no HTTP overhead per message)
  + Binary data support
  - Complex connection management
  - Harder to scale (sticky sessions or Redis pub/sub)
  - Proxies need special configuration
  
  Best for: chat, gaming, collaborative editing, trading
```

### SSE Implementation
```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from sse_starlette.sse import EventSourceResponse
import asyncio

app = FastAPI()


@app.get("/events/notifications/{user_id}")
async def notification_stream(user_id: str, request: Request):
    async def event_generator():
        pubsub = app.state.redis.pubsub()
        await pubsub.subscribe(f"notifications:{user_id}")
        
        try:
            while True:
                # Check if client disconnected
                if await request.is_disconnected():
                    break
                
                message = await pubsub.get_message(
                    ignore_subscribe_messages=True, timeout=1.0
                )
                
                if message:
                    yield {
                        "event": "notification",
                        "data": message["data"].decode(),
                    }
                else:
                    # Send keepalive every 15s
                    yield {"event": "ping", "data": ""}
                    await asyncio.sleep(15)
        finally:
            await pubsub.unsubscribe(f"notifications:{user_id}")
    
    return EventSourceResponse(event_generator())


# Trigger notification from anywhere in the app
async def send_notification(user_id: str, title: str, body: str):
    import json
    payload = json.dumps({"title": title, "body": body})
    await app.state.redis.publish(f"notifications:{user_id}", payload)


# Usage: order status updates
@app.post("/orders/{order_id}/ship")
async def ship_order(order_id: str):
    order = await OrderService.mark_shipped(order_id)
    
    # Push real-time notification
    await send_notification(
        str(order.user_id),
        "Order Shipped!",
        f"Order #{order_id[:8]} is on its way",
    )
    
    return order
```

---

## 5. Live Dashboard (Order Tracking)

```python
@app.get("/events/orders/{order_id}/track")
async def track_order(order_id: str, request: Request):
    """Real-time order status updates via SSE."""
    
    async def generate():
        pubsub = app.state.redis.pubsub()
        await pubsub.subscribe(f"order_updates:{order_id}")
        
        # Send current status immediately
        order = await OrderService.get(order_id)
        yield {
            "event": "status",
            "data": json.dumps({
                "status": order.status,
                "updated_at": order.updated_at.isoformat(),
                "location": order.current_location,
            }),
        }
        
        try:
            while True:
                if await request.is_disconnected():
                    break
                
                message = await pubsub.get_message(
                    ignore_subscribe_messages=True, timeout=1.0
                )
                if message:
                    yield {"event": "status", "data": message["data"].decode()}
                
                await asyncio.sleep(1)
        finally:
            await pubsub.unsubscribe(f"order_updates:{order_id}")
    
    return EventSourceResponse(generate())


# Client-side JavaScript
"""
const eventSource = new EventSource('/events/orders/abc123/track');

eventSource.addEventListener('status', (event) => {
    const data = JSON.parse(event.data);
    updateStatusUI(data.status);
    updateLocationOnMap(data.location);
});

eventSource.onerror = () => {
    // Auto-reconnects (browser built-in)
    console.log('Connection lost, reconnecting...');
};
"""
```

---

## 6. WebSocket Authentication

```python
from fastapi import WebSocket, WebSocketDisconnect, Query, status


async def authenticate_ws_token(token: str):
    """Validate JWT token for WebSocket connection."""
    from app.core.auth import verify_token
    from app.database import async_session
    from app.models.user import User
    
    payload = verify_token(token, "access")
    
    async with async_session() as db:
        user = await db.get(User, payload.sub)
        if not user or not user.is_active:
            raise ValueError("Invalid user")
        return user


@app.websocket("/ws/{channel}")
async def websocket_with_auth(
    websocket: WebSocket,
    channel: str,
    token: str = Query(None),
):
    # Method 1: Token in query param
    if not token:
        await websocket.close(code=status.WS_1008_POLICY_VIOLATION)
        return
    
    try:
        user = await authenticate_ws_token(token)
    except Exception:
        await websocket.close(code=4001)
        return
    
    # Authenticated — proceed
    await websocket.accept()
    # ...


# Method 2: Token in first message (more secure — not in URL/logs)
@app.websocket("/ws/secure/{channel}")
async def websocket_first_message_auth(websocket: WebSocket, channel: str):
    await websocket.accept()
    
    # Wait for auth message (timeout after 5s)
    try:
        data = await asyncio.wait_for(websocket.receive_text(), timeout=5.0)
        auth_msg = json.loads(data)
        
        if auth_msg.get("type") != "auth":
            await websocket.close(code=4001)
            return
        
        user = await authenticate_ws_token(auth_msg["token"])
        await websocket.send_json({"type": "auth_success", "user_id": str(user.id)})
        
    except (asyncio.TimeoutError, Exception):
        await websocket.close(code=4001)
        return
    
    # Now handle normal messages...
    try:
        while True:
            data = await websocket.receive_text()
            # Process authenticated messages
    except WebSocketDisconnect:
        pass
```

---

## 7. Interview Questions

### Q: How would you implement real-time notifications in FastAPI?
```
Architecture:
  Event Source → Redis Pub/Sub → FastAPI SSE/WebSocket → Client

Flow:
  1. Order shipped → publish to Redis channel "notifications:{user_id}"
  2. FastAPI SSE endpoint subscribes to that channel
  3. Client has EventSource connected to /events/notifications
  4. Message flows: Redis → FastAPI → Browser (instant)

Why Redis Pub/Sub?
  - Works across multiple server instances
  - Fire-and-forget (no persistence needed for notifications)
  - Sub-millisecond delivery
  - If persistence needed: use Redis Streams instead

Scaling:
  - SSE: each connection is a long-lived HTTP response
  - 10,000 users = 10,000 open connections
  - Nginx: increase worker_connections, disable buffering
  - Use separate notification service if > 100K connections
```

### Q: WebSocket vs SSE — when to use which?
```
Use SSE when:
  - Server pushes updates to client (one-way)
  - Notifications, live feeds, progress bars
  - Need auto-reconnect (browser handles it)
  - Want simple HTTP infrastructure (no sticky sessions)

Use WebSocket when:
  - Need bidirectional communication
  - Chat, multiplayer games, collaborative editing
  - Need binary data (file transfer)
  - Low-latency requirement (no HTTP overhead)

Hybrid (common in production):
  - SSE for server → client (notifications, status updates)
  - REST POST for client → server (send message, update status)
  - Simpler than full WebSocket, works with standard load balancers
```
