# WebSocket・リアルタイム通信

HTTP は「クライアントが要求してサーバーが返す」一方向の通信ですが、WebSocket は**サーバーとクライアントが双方向にいつでもメッセージを送り合える**接続です。チャット・通知・リアルタイムダッシュボード・ゲームなど、サーバーからプッシュが必要な場面で使います。

---

## はじめて読む人へ

「X（旧 Twitter）のタイムラインが自動で更新される」「チャットアプリでメッセージが即座に届く」これらはサーバーからのプッシュが必要です。通常の HTTP では「クライアントが聞かない限りサーバーは何も送れない」のでリアルタイムに弱いです。

### 読む前に押さえること

- [ネットワーク基礎](ネットワーク基礎.md) の HTTP リクエスト・レスポンスの概念
- [FastAPI](FastAPI.md) の基本的なエンドポイント実装

### 読み終えたら説明できること

- ポーリング・SSE・WebSocket の違いとそれぞれの使いどころを説明できる
- FastAPI で WebSocket サーバーとブラウザクライアントを実装できる
- 接続管理（ルームへの参加・離脱・ブロードキャスト）の仕組みを説明できる

---

## 3 つのリアルタイム手法

1. **ポーリング（Polling）**：クライアントが 1 秒ごとに「新しいメッセージある？」と問い合わせ、サーバーが「ありません」「あります！」と返す。シンプルだが無駄なリクエストが多い。
2. **SSE（Server-Sent Events）**：クライアントが接続を確立し、サーバーが更新があったときだけデータを送信する。サーバー → クライアントの一方向のみ。
3. **WebSocket**：クライアントが接続（HTTP → WebSocket にアップグレード）し、サーバーとクライアントがいつでも双方向にメッセージを送れる。チャット・ゲームなど双方向が必要な場面に最適。
| 手法 | 方向 | 接続 | 向いているケース |
|------|------|------|----------------|
| ポーリング | ← | 毎回 HTTP | シンプルな更新チェック |
| SSE | サーバー→クライアント | 1 本の持続接続 | 通知・ライブフィード・進捗表示 |
| WebSocket | 双方向 | 持続接続 | チャット・ゲーム・コラボ編集 |

---

## WebSocket のハンドシェイク

WebSocket は最初だけ HTTP でネゴシエーションし、その後 TCP 接続をそのまま WebSocket に切り替えます。

| クライアント → サーバー（HTTP リクエスト） | サーバー → クライアント（101 Switching Protocols） |
|---|---|
| GET /ws HTTP/1.1 | HTTP/1.1 101 Switching Protocols |
| Upgrade: websocket | Upgrade: websocket |
| Connection: Upgrade | Connection: Upgrade |
| Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ== | 以降: TCP 接続をそのまま使って双方向通信 |
---

## FastAPI で WebSocket サーバー

### 基本（1 対 1 通信）

```python
# main.py
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    await websocket.accept()           # 接続を受け入れる
    print(f"接続: {client_id}")

    try:
        while True:
            data = await websocket.receive_text()   # メッセージを受信
            echo = f"[{client_id}] {data}"
            await websocket.send_text(echo)         # 送り返す
    except WebSocketDisconnect:
        print(f"切断: {client_id}")
```

### 接続管理（チャットルーム）

複数クライアントへのブロードキャストには、接続を管理するクラスを作ります。

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import defaultdict

app = FastAPI()

class ConnectionManager:
    def __init__(self):
        # ルームID → 接続リスト
        self.rooms: dict[str, list[WebSocket]] = defaultdict(list)

    async def join(self, room: str, ws: WebSocket):
        await ws.accept()
        self.rooms[room].append(ws)

    def leave(self, room: str, ws: WebSocket):
        self.rooms[room].remove(ws)

    async def broadcast(self, room: str, message: str, sender: WebSocket = None):
        """ルーム内の全員（送信者を除く）にメッセージを送る"""
        for ws in self.rooms[room]:
            if ws != sender:
                await ws.send_text(message)

manager = ConnectionManager()

@app.websocket("/chat/{room}/{username}")
async def chat(websocket: WebSocket, room: str, username: str):
    await manager.join(room, websocket)
    await manager.broadcast(room, f"✅ {username} が入室しました")

    try:
        while True:
            msg = await websocket.receive_text()
            await manager.broadcast(room, f"{username}: {msg}", sender=websocket)
    except WebSocketDisconnect:
        manager.leave(room, websocket)
        await manager.broadcast(room, f"👋 {username} が退室しました")
```

---

## JavaScript クライアント

```html
<!-- chat.html -->
<!DOCTYPE html>
<html>
<body>
  <div id="messages"></div>
  <input id="input" placeholder="メッセージを入力">
  <button onclick="send()">送信</button>

  <script>
    const ws = new WebSocket("ws://localhost:8000/chat/room1/田中");
    const messages = document.getElementById("messages");

    // サーバーからメッセージを受け取る
    ws.onmessage = (event) => {
      const p = document.createElement("p");
      p.textContent = event.data;
      messages.appendChild(p);
    };

    ws.onopen  = () => console.log("接続しました");
    ws.onclose = () => console.log("切断されました");
    ws.onerror = (e) => console.error("エラー:", e);

    function send() {
      const input = document.getElementById("input");
      ws.send(input.value);  // サーバーにメッセージを送る
      input.value = "";
    }
  </script>
</body>
</html>
```

---

## SSE（Server-Sent Events）

サーバー → クライアントへの**一方向**プッシュが必要なだけなら SSE が軽量でシンプルです。

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio

app = FastAPI()

async def event_generator():
    for i in range(100):
        yield f"data: カウント {i}\n\n"   # SSE フォーマット: "data: ...\n\n"
        await asyncio.sleep(1)

@app.get("/events")
async def sse_endpoint():
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
    )
```

```javascript
// JavaScript クライアント（SSE）
const es = new EventSource("/events");
es.onmessage = (event) => {
  console.log(event.data);  // "カウント 0", "カウント 1", ...
};
```

---

## React との統合

```tsx
// hooks/useWebSocket.ts
import { useEffect, useRef, useState } from "react";

export function useWebSocket(url: string) {
  const wsRef = useRef<WebSocket | null>(null);
  const [messages, setMessages] = useState<string[]>([]);
  const [status, setStatus] = useState<"connecting" | "open" | "closed">("connecting");

  useEffect(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen  = () => setStatus("open");
    ws.onclose = () => setStatus("closed");
    ws.onmessage = (e) => setMessages((prev) => [...prev, e.data]);

    return () => ws.close();   // コンポーネントのアンマウント時に切断
  }, [url]);

  const send = (msg: string) => wsRef.current?.send(msg);

  return { messages, status, send };
}

// コンポーネントでの使い方
function ChatRoom({ room, username }: { room: string; username: string }) {
  const { messages, status, send } = useWebSocket(
    `ws://localhost:8000/chat/${room}/${username}`
  );

  return (
    <div>
      <p>状態: {status}</p>
      <ul>{messages.map((m, i) => <li key={i}>{m}</li>)}</ul>
      <button onClick={() => send("こんにちは")}>送信</button>
    </div>
  );
}
```

---

## 確認問題

1. HTTP のポーリングと WebSocket の違いを「接続の維持」という観点から説明してください。
2. WebSocket と SSE はどう使い分けますか？具体的なユースケースを 1 つずつ挙げてください。
3. `ConnectionManager` の `broadcast` メソッドで `sender` を除外している理由は何ですか？

---

## 関連ページ

- [FastAPI](FastAPI.md) — WebSocket サーバーの実装基盤
- [ネットワーク基礎](ネットワーク基礎.md) — HTTP・TCP の基礎
- [GraphQL](GraphQL.md) — Subscription（WebSocket ベースのリアルタイム）
- [React](React.md) — WebSocket フックをコンポーネントで使う
