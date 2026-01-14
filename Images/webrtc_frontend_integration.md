# Интеграция WebRTC-стриминга для фронтенда

Документ описывает протокол работы фронтенда с сервисом стриминга по WebSocket:
какие события отправлять, какие получать, в каком порядке и с какими payload’ами.

Роли:

- **WEB** — веб-клиент, создаёт комнату и смотрит стрим.
- **DEVICE** — устройство, публикует медиа (камера/микрофон) по приглашению.

Все примеры ниже даны для браузерного JavaScript, но протокол одинаков для любых клиентов.

---

## 1. Подключение к WebSocket

### 1.1. URL

```text
ws://<host>:<port>/ws/connect/<ROLE>/<client_id>/<token>
```

Где:

- `ROLE` — `"WEB"` или `"DEVICE"`;
- `client_id` — произвольный идентификатор клиента (случайное число или UUID);
- `token` — auth-токен (для DEV может быть заглушка).

**Пример (WEB):**

```js
const clientId = Math.floor(Math.random() * 1000);
const ws = new WebSocket(`ws://127.0.0.1:8002/ws/connect/WEB/${clientId}/some_web_token`);
```

**Пример (DEVICE):**

```js
const clientId = Math.floor(Math.random() * 1000);
const ws = new WebSocket(`ws://127.0.0.1:8002/ws/connect/DEVICE/${clientId}/dummy_token`);
```

---

## 2. Общий формат сообщений

Любое сообщение имеет вид:

```json
{
  "event": "<event_name>",
  "data": { ... }
}
```

Обработка на фронте:

```js
ws.onmessage = (e) => {
  const msg = JSON.parse(e.data);
  console.log("Got", msg.event, "event", msg.data);
};
```

---

## 3. Общие вещи для WebRTC

### 3.1. RTCPeerConnection

Обе стороны (WEB и DEVICE) создают свой `RTCPeerConnection`:

```js
const pc = new RTCPeerConnection({
  iceServers: [
    { urls: "stun:stun.l.google.com:19302" },
    {
      urls: "turn:relay1.expressturn.com:3480",
      username: "…",
      credential: "…"
    }
  ]
});
```

### 3.2. ICE-кандидаты

Направление: **клиент ⇄ сервер**.

**От клиента к серверу:**

```js
pc.onicecandidate = (e) => {
  ws.send(JSON.stringify({
    event: "ice",
    data: e.candidate || {}
  }));
};
```

Payload:

```json
{
  "event": "ice",
  "data": {
    "candidate": "candidate:...",
    "sdpMid": "0",
    "sdpMLineIndex": 0,
    "typ": "sub"
  }
}
```

`sub` передается для WEB клиента, как получателя данных (supscriber)
`pub` для DEVICE как для отправителя данных (publisher)

**От сервера к клиенту:**

Приходят в SDP

### 3.3. Получение удалённых треков

WEB-клиент:

```js
subscriberPC.ontrack = (event) => {    
    if (event.streams && event.streams[0]) {
        const videoElement = createRemoteVideo(message.data.feed);
        videoElement.srcObject = event.streams[0];
        videoElement.play().catch(e => {
            console.log('Play blocked, need user interaction');
        });
    }
};
```

Создание DOM-элемента под видео:

```js
function createRemoteVideo(streamId) {
    if (!document.getElementById(`remote_${streamId}`)) {
        const videoContainer = document.createElement('div');
        videoContainer.classList = ["videoContainer"];
        
        const video = document.createElement('video');
        video.id = `remote_${streamId}`;
        video.classList = ["video", "remote-video"];
        video.autoplay = true;
        video.playsinline = true;
        
        const label = document.createElement('div');
        label.classList = ["label"];
        label.textContent = `Remote Stream ${streamId}`;
    
        videoContainer.appendChild(video);
        videoContainer.appendChild(label);
        remoteVideos.appendChild(videoContainer);
    
        return video;
    } else {
        return document.getElementById(`remote_${streamId}`)
    }
}
```

---

## 4. Флоу для WEB (создание комнаты и просмотр стрима)

### 4.1. Подключение к WebSocket

1. Создать `client_id`.
2. Открыть `ws://…/ws/connect/WEB/<client_id>/<token>`.
3. Навесить обработчик `onmessage`.

---

### 4.2. Создание комнаты
Входная точка для стриминга. При нажатии кноки на UI должен выполняться этот запрос

**Запрос от WEB:**

```json
{
  "event": "create_room",
  "data": {}
}
```

**Ответ от сервера:**

```json
{
  "event": "room_created",
  "data": {
    "room_id": 166
  }
}
```

WEB должен сохранить `room_id`.

---

### 4.3. Подключение к комнате как publisher

WEB подключается к комнате как «publisher»-участник, но при этом может не отдавать медиа — цель: иметь хендл для событий комнаты.

**Запрос от WEB:**

```json
{
  "event": "join_as_publisher",
  "data": {
    "room_id": 166,
    "inviter": <client_id>
  }
}
```

**Ответ от сервера:**

```json
{
  "event": "joined_as_publisher",
  "data": {
    "room_id": 166
  }
}
```

После этого WEB может приглашать устройства.

---

### 4.4. Приглашение устройств

**Запрос от WEB:**

```json
{
  "event": "invite_devices",
  "data": {
    "room_id": <currentRoomID>
  }
}
```

**Что делает сервер:**

- Находит доступные `DEVICE`.
- Каждому DEVICE отправляет событие `start_streaming` (см. раздел 5).

---

### 4.5. Получение информации о новом стриме

После того как DEVICE подключился к комнате и начал публиковать медиа, сервер передаёт WEB-клиенту событие:

```json
{
  "event": "start_listening",
  "data": {
    "feed": <feedID>,
  }
}
```

WEB должен:

1. Создать `RTCPeerConnection` для роли подписчика.
2. Выполнить флоу подписки (см. ниже).

---

### 4.6. Подписка на стрим (WEB как subscriber)

#### 4.6.1. JOIN как subscriber

**Запрос от WEB:**

```json
{
  "event": "join_as_subscriber",
  "data": {
    "room_id": <currentRoomID>,
    "feed": <feedID>
  }
}
```

Дальше сервер, после attach к Janus, присылает SDP-offer.

#### 4.6.2. Получение SDP-offer

**Сервер → WEB:**

```json
{
  "event": "offer_to_subscriber",
  "data": {
    "type": "offer",
    "sdp": "<remote_sdp>"
  }
}
```

WEB создаёт answer и отправляет его через событие `start_receiving`.

```js
if (msg.event === "offer_to_subscriber") {
    // Создать PC для подписчика  
    const subscriberPC = new RTCPeerConnection({
        iceServers: iceServers,
        bundlePolicy: 'max-bundle',
        rtcpMuxPolicy: 'require'
    });
    // Добавить событие on_track
    subscriberPC.ontrack = (event) => {
        if (event.streams && event.streams[0]) {
            const videoElement = createRemoteVideo(message.data.feed);
            videoElement.srcObject = event.streams[0];
            videoElement.play().catch(e => {console.log('Play blocked, need user interaction')});
        }
    };
    // Set up ICE candidate handler
    subscriberPC.onicecandidate = async(e) => {
        if (e.candidate) {
            await send_ice_candidate(e.candidate, "sub");
        } else {
            console.log("Subscriber ICE gathering complete");
            await send_ice_candidate(null, "sub");
        }
    };
    // Set remote description
    const remoteDesc = new RTCSessionDescription({
        type: 'offer',
        sdp: message.data.sdp
    });
    await subscriberPC.setRemoteDescription(remoteDesc);
    const answer = await subscriberPC.createAnswer();
    const lines = answer.sdp.split('\n');
    lines.forEach(line => {
        if (line.includes('sendonly') || line.includes('recvonly') || 
            line.includes('sendrecv') || line.includes('inactive')) {
            console.log("  ", line);
        }
    });
    await subscriberPC.setLocalDescription(answer);
    ws.send(JSON.stringify({
      event: "start_receiving", 
      data: {
          sdp: answer.sdp,
          feed: message.data.feed  // Include feed ID
      }
    }));
}
```

После этого:

- Janus начинает посылать медиа.
- На WEB вызывается `pc.ontrack`, приходят `audio` и `video` треки.
- Браузер начинает воспроизведение удалённого стрима.

---

## 5. Флоу для DEVICE (публикация медиа)

### 5.1. Подключение к WebSocket

1. Создать `client_id`.
2. Открыть `ws://…/ws/connect/DEVICE/<client_id>/<token>`.
3. Навесить `onmessage`.

---

### 5.2. Получение приглашения на стрим

**Сервер → DEVICE:**

```json
{
  "event": "start_streaming",
  "data": {
    "room_id": 166,
    "inviter": "123"
  }
}
```

JS:

```js
if (msg.event === "start_streaming") {
  const { room_id, inviter } = msg.data;
  await joinRoom(room_id, inviter);
}
```

---

### 5.3. Подключение к комнате как publisher

```js
async function joinRoom(room_id, inviter) {
    console.log("Joining a room")

    stop_btn.disabled = false;

    localStream = await set_local_stream();
    pc = await create_pc();
    pc = await addLocalTracksToPC();

    pc.onicecandidate = async(e) => on_ice_candidate_generated(e);
    
    console.log("Sending join room event to WS")
    ws.send(JSON.stringify({event: "join_as_publisher", data: {room_id: room_id, inviter: inviter}}))
}
```

**Ответ от сервера:**

```json
{
  "event": "joined_as_publisher",
  "data": {
    "room_id": 166
  }
}
```

После этого DEVICE должен сделать WebRTC-offer и отправить его в сервис.

---

### 5.4. Отправка SDP-offer от DEVICE

```js
async function sendOffer() {
    console.log("Sending an offer")
    offer = await pc.createOffer();
    pc = await set_local_description();
    console.log("Sending offer to WS")
    ws.send(JSON.stringify({event: "offer_from_device", data: {sdp: offer.sdp}}))
}
```

После конфигурации Janus возвращает answer.

---

### 5.5. Получение SDP-answer от сервиса

**Сервер → DEVICE:**

```json
{
  "event": "answer_to_client",
  "data": {
    "type": "answer",
    "sdp": "<remote_sdp>"
  }
}
```

JS:

```js
if (msg.event === "answer_to_client") {
  const remoteDesc = new RTCSessionDescription({
    type: "answer",
    sdp: msg.data.sdp
  });

  await pc.setRemoteDescription(remoteDesc);
}
```

После этого медиапоток с DEVICE публикуется в комнате, а WEB-клиент может на него подписаться (см. раздел 4.5–4.6).

---

## 6. Завершение сессии

### 6.1. Остановка публикации (DEVICE)

```json
{
  "event": "unpublish",
  "data": {}
}
```

Сервис снимает паблишера в Janus и рассылает события о завершении стрима.

### 6.2. Выход из комнаты

И WEB, и DEVICE могут отправить:

```json
{
  "event": "leave",
  "data": {}
}
```

## 7. Ошибки

Любая ошибка протокола или служебная ошибка возвращается событием:

```json
{
  "event": "error",
  "data": {
    "code": "<string|int>",
    "message": "<human_readable_message>"
  }
}
```

Клиент должен логировать ошибку и по необходимости показывать пользователю.

Пример обработки:

```js
if (msg.event === "error") {
  console.error("Error event:", msg.data);
}
```

---

## 8. Краткое резюме потоков

### WEB

1. Подключиться к WS как `WEB`.
2. `create_room` → получить `room_created.room_id`.
3. `join_as_publisher` (для событий комнаты).
4. `invite_devices`.
5. Ждать `start_listening(feed)`.
6. `join_as_subscriber(room_id, feed)`.
7. Ждать `offer_to_subscriber` → `setRemoteDescription(offer)`.
8. `createAnswer()` → `setLocalDescription(answer)` → `start_receiving({ sdp })`.
9. Слушать `pc.ontrack` и показывать видео.

### DEVICE

1. Подключиться к WS как `DEVICE`.
2. Ждать `start_streaming(room_id, inviter)`.
3. Получить media (`getUserMedia`), создать `RTCPeerConnection`, добавить треки.
4. `join_as_publisher(room_id, inviter)`.
5. Ждать `joined_as_publisher`.
6. `createOffer()` → `setLocalDescription()` → `offer_from_device({ sdp })`.
7. Ждать `answer_to_client` → `setRemoteDescription(answer)`.
8. Стрим публикуется в комнату; WEB подписывается по feed.
