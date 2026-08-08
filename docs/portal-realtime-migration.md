# Handoff: replacing the hand-rolled realtime layer with Portal

**Status:** proposal, not implemented. Nothing in this PR changes runtime behavior — it adds
this document only.

**Audience:** whoever picks up the realtime work on BioShield.

**Scope:** the ESP32 firmware is not modified. The FastAPI backend keeps every endpoint it
has today. What changes is who fans data out to browsers.

---

## 1. TL;DR

Today the backend does double duty: it talks to devices *and* it hand-maintains the
browser fan-out — connection registries, per-user subscriptions, and last-known state, all
in process memory. That second job is the one that does not survive a restart or a second
worker.

[Portal](https://docs.useportal.co) replaces exactly that second job. The proposal:

- FastAPI stays the bridge and the only holder of the Portal secret key.
- Device telemetry is throttled, then published to a Portal channel over HTTPS.
- The browser subscribes to that channel with `useChannel()` and stops running its own
  WebSocket client.
- Motor commands keep their current, short path and deliberately **do not** travel
  through Portal.

Net deletion: ~230 lines of connection bookkeeping, replaced by roughly 60 lines of bridge
code.

---

## 2. Why change anything

Three findings from reading the current code. All three are verifiable in the tree, not
opinions about style.

**2.1 — All shared state is per-process memory.**

`Backend/app/utils/WsManager.py` builds a singleton over five plain dicts:

```python
esp_connections      # device_id -> WebSocket
frontend_connections # user_id   -> WebSocket
esp_states           # device_id -> last known reading
user_devices         # user_id   -> {device_id}
device_subscribers   # device_id -> {user_id}
```

`ConnectionManager.__new__` makes it a singleton *within one Python process*. Run
`uvicorn --workers 2` and there are two independent registries: a device connected to
worker A is invisible to a browser served by worker B, and `POST /api/esp/{id}/motor` will
answer `404 ESP no conectado` roughly half the time.

**2.2 — Nothing is persisted, despite the code that was written for it.**

- `update_esp_data()` is defined at `Backend/app/routes/esp_socket.py:38` and is **never
  called** from anywhere in the tree. `Esp.json_sensores` is never written.
- `Backend/app/utils/BufferManager.py` implements a complete disk buffer (140 lines) and
  its only import is commented out at `Backend/app/utils/WsManager.py:9`.

So `esp_states` is the only record of device state that exists, and it dies with the
process.

**2.3 — The late-joiner problem was already solved by hand.**

`Backend/app/routes/esp_socket.py:235-241` sends the current state to a browser right after
it subscribes, so the dashboard never renders empty. That instinct is correct and Portal
has a first-class version of it. Worth noting because the migration preserves the behavior
rather than introducing it.

---

## 3. Target architecture

```
╔═══════════════════════════════════════════════════════════════════════╗
║  LAN                                                                  ║
║    ┌──────────────────┐                                               ║
║    │      ESP32       │   firmware UNCHANGED                          ║
║    │  DHT11 · stepper │   WebSocketsClient · auto-reconnect           ║
║    └────────┬─────────┘                                               ║
║             │  ws://<lan-ip>:8000/ws/esp/{chipId}                     ║
║   SENSOR_DATA ↑ every 2 s       ↓ MOTOR_COMMAND                       ║
║             │                                                         ║
║    ┌────────▼────────────────────────────────────────────┐            ║
║    │  FastAPI · THE BRIDGE · sole holder of the sk_      │            ║
║    │                                                     │            ║
║    │   ① /ws/esp/{id}         receive → throttle → publish│           ║
║    │   ② /api/esp/{id}/motor  command → device (unchanged)│           ║
║    │   ③ /api/portal/token    mint the user's Portal JWT  │           ║
║    └────────┬───────────────────────────────┬────────────┘            ║
╚═════════════│═══════════════════════════════│═════════════════════════╝
              │ ① HTTPS · Bearer sk_          │ ③ POST /v1/tokens
              │   server publish              │   Bearer sk_
              ▼                               ▼
     ┌────────────────────────────────────────────────────┐
     │  PORTAL                                            │
     │  channel   bio-{chipId}-{YYYYMMDDHH}  ← hourly roll │
     └──────────────────────┬─────────────────────────────┘
                            │ WebSocket + user JWT
                            ▼
     ┌────────────────────────────────────────────────────┐
     │  esp-monitor · React 18 + Vite                     │
     │  useChannel()        ·      websocket.js DELETED    │
     └────────────────────────────────────────────────────┘
```

A Portal channel is **a name, not a place**. Nobody hosts it. The publisher (FastAPI, over
HTTPS with the secret key) and the subscriber (the browser, over a WebSocket with a user
JWT) never meet and share no process. That is why the frontend could be swapped for Next.js,
another Vite app, or the Capacitor Android build without touching anything above.

### The three paths, and why they differ

```
  telemetry   ESP32 ─▶ FastAPI ─▶ Portal ─▶ dashboard     3 hops
  command     dashboard ─▶ FastAPI ─▶ ESP32               2 hops, NO Portal
  echo        ESP32 ─▶ FastAPI ─▶ Portal ─▶ dashboard     confirms real state
```

**The command path deliberately bypasses Portal.** `STOP_MOTOR` sets `emergencyStop` in the
firmware (`Esp32/station_main/station_main.ino:309`); an emergency stop should not gain
network hops. `POST /api/esp/{device_id}/motor` stays exactly as it is today.

The consequence is worth stating explicitly: what the dashboard renders as "motor running"
is **not** local optimism. It is the `MOTOR_STATUS` frame the device itself sent back up
the telemetry path. The command path is fire-and-forget; the echo path is the source of
truth.

---

## 4. Design decisions

### 4.1 The publish throttle is load-bearing

The firmware sends every 2 s (`Esp32/station_main/station_main.ino:346`) — 1800 messages
per hour. Portal channels stop delivering somewhere in the low hundreds of persistent
messages (see §6). Published naively, a channel dies in about six minutes.

Publish on **change**, not on a timer. The thresholds come from the sensor: a DHT11 is
±2 °C and ±5 % RH accurate with 1 °C / 1 % resolution, so anything finer than the values
below is publishing noise.

```python
TEMP_THRESHOLD_C  = 1.0   # below this is DHT11 noise
HUM_THRESHOLD_PCT = 3.0
HEARTBEAT_S       = 60    # publish anyway, so a late joiner sees something
MIN_INTERVAL_S    = 2     # hard ceiling on publish rate
```

In a stable room this drops ~1800/h to ~60/h.

Motor lifecycle events (`RUNNING`, `STOPPED`, `COMPLETED`) bypass the throttle — they are
rare and each one means something. The position update every 128 steps
(`Esp32/station_main/station_main.ino:228`) does not: rate-limit it to roughly one every
1.5 s.

### 4.2 Channel rotation

Channel id: `bio-{chipId}-{YYYYMMDDHH}`, computed in **UTC** on both sides so backend and
frontend never disagree at a boundary.

With the throttle above, an hour of running uses well under half the budget. Rotation exists
so an unattended device does not silently fall off after a few hours, and it also mitigates
the stall described in §6.2.

> For a short live demo (under ~30 minutes) rotation can be skipped and the channel id
> reduced to `bio-{chipId}`. Skipping it is a legitimate simplification; forgetting it is
> not. Decide deliberately.

### 4.3 Surface the budget in the UI

Show messages-published-this-channel in the dashboard. Two reasons:

1. It makes a real platform limit visible instead of letting it fail silently.
2. Per §6.2, a stalled Portal channel is indistinguishable from an idle one from the
   client's side. The dashboard therefore also needs a **staleness indicator** — time since
   the last message — because "no new data" cannot be trusted to mean "nothing is
   happening".

### 4.4 Authorization moves from subscribe-time to mint-time

Today the user↔device access check runs on every `SUBSCRIBE`
(`Backend/app/routes/esp_socket.py:213-222`). With Portal it runs **once, when the token is
minted**. Same query, same models, no new authorization logic:

```python
devices = (
    db.query(Usuario_Esp)
      .join(Esp).join(User)
      .filter(User.name == user.name)
      .all()
)
# those device ids become the token's `channels` grants
```

The browser only needs `connect`, never `publish` — it does not publish to Portal at all,
since commands go through FastAPI. Least privilege comes free.

### 4.5 Initial state for a late joiner

Use `useChannel({ channelId, history: 1 })`. The last message arrives before first render,
so the dashboard never starts blank. No Portal config deploy required.

A more durable option — a Portal channel extension with `transport: "http"` and
`onSnapshot()`, which keeps current state inside Portal and survives a FastAPI restart — is
described in §9 as a follow-up. Not needed for a working demo.

---

## 5. Implementation plan

| File | Change |
|---|---|
| `Esp32/station_main/station_main.ino` | none (but see §7.1 — rotate the committed Wi-Fi password) |
| `Backend/app/utils/WsManager.py` | drop `frontend_connections`, `user_devices`, `device_subscribers`. Keep `esp_connections`, `is_connected_esp`, `send_command_to_esp` |
| `Backend/app/routes/esp_socket.py` | delete `/ws/frontend` (~80 lines). Replace the `broadcast_esp_data()` call with `publish_to_portal()` |
| `Backend/app/utils/portal_client.py` | **new** — publish + mint helpers |
| `Backend/app/routes/portal_routes.py` | **new** — `POST /api/portal/token` |
| `esp-monitor/src/services/websocket.js` | **delete** (144 lines) |
| `esp-monitor/src/components/Dashboard.jsx` | `useChannel()` instead of `WebSocketService` |
| `esp-monitor/src/App.jsx` | wrap in `PortalProvider` |
| `.env` | add `PORTAL_SECRET_KEY`; frontend gets `VITE_PORTAL_KEY` |

Two things already line up: `httpx==0.27.2` is in `requirements.txt`, so the backend needs
no new dependency to call Portal; and `react@^18.3.1` satisfies `@portalsdk/react`'s peer
range (`>=18 <20`).

### 5.1 Backend — the Portal client

```python
# Backend/app/utils/portal_client.py
import os
from datetime import datetime, timezone

import httpx

PORTAL_API = "https://api.useportal.co"
SECRET_KEY = os.environ["PORTAL_SECRET_KEY"]  # sk_... — server only, never in a browser


def channel_id(chip_id: str) -> str:
    """Hourly channel, UTC so backend and frontend always agree."""
    stamp = datetime.now(timezone.utc).strftime("%Y%m%d%H")
    return f"bio-{chip_id}-{stamp}"


async def publish(chip_id: str, message_type: str, content: dict) -> dict:
    """Server publish. Pre-trusted: bypasses channel authz and publish middleware."""
    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.post(
            f"{PORTAL_API}/v1/channels/{channel_id(chip_id)}/messages",
            headers={"Authorization": f"Bearer {SECRET_KEY}"},
            json={
                "senderId": chip_id,      # required on the server path
                "type": message_type,     # no dots: a dotted prefix routes to an extension
                "content": content,       # opaque to Portal, capped at 2 KB
            },
        )
        response.raise_for_status()
        return response.json()            # {"id", "seq", "timestamp"}


async def mint_token(user_id: str, channel_ids: list[str], ttl: str = "1h") -> dict:
    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.post(
            f"{PORTAL_API}/v1/tokens",
            headers={"Authorization": f"Bearer {SECRET_KEY}"},
            json={
                "userId": user_id,
                "channels": {cid: ["connect"] for cid in channel_ids},
                "ttl": ttl,
            },
        )
        response.raise_for_status()
        return response.json()            # {"token", "expiresAt"}
```

> `type` must not contain a dot. Portal routes dotted prefixes to channel extensions, and
> with no extension attached the message never reaches the channel. This cost us hours on
> another project — use `sensor-data`, not `sensor.data`.

### 5.2 Backend — the throttle

```python
# Backend/app/utils/portal_throttle.py
import time

TEMP_THRESHOLD_C = 1.0
HUM_THRESHOLD_PCT = 3.0
HEARTBEAT_S = 60
MIN_INTERVAL_S = 2

_last: dict[str, dict] = {}   # chip_id -> {"t": epoch, "temperature": x, "humidity": y}


def should_publish(chip_id: str, reading: dict, now: float | None = None) -> bool:
    now = now if now is not None else time.time()
    previous = _last.get(chip_id)

    if previous is None:
        _last[chip_id] = {**reading, "t": now}
        return True

    elapsed = now - previous["t"]
    if elapsed < MIN_INTERVAL_S:
        return False

    changed = (
        elapsed >= HEARTBEAT_S
        or abs(reading["temperature"] - previous["temperature"]) >= TEMP_THRESHOLD_C
        or abs(reading["humidity"] - previous["humidity"]) >= HUM_THRESHOLD_PCT
    )
    if changed:
        _last[chip_id] = {**reading, "t": now}
    return changed
```

Wire it where `broadcast_esp_data()` is called today
(`Backend/app/routes/esp_socket.py:82`). Motor events call `publish()` directly and skip
`should_publish()`.

### 5.3 Backend — the token route

```python
# Backend/app/routes/portal_routes.py
from datetime import datetime, timedelta, timezone

from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from app.database.modelsDB import Esp, User, Usuario_Esp
from app.utils.database_dependencies import get_transactional_db
from app.utils.portal_client import mint_token

portal_routes = APIRouter()


@portal_routes.post("/api/portal/token")
async def portal_token(user=Depends(...), db: Session = Depends(get_transactional_db)):
    """Mint a Portal JWT scoped to exactly the channels this user's devices publish to."""
    rows = (
        db.query(Esp.identification)
          .join(Usuario_Esp).join(User)
          .filter(User.name == user.name)
          .all()
    )

    # Grant the current hour and the next one, so a channel roll mid-session is seamless.
    now = datetime.now(timezone.utc)
    stamps = [
        now.strftime("%Y%m%d%H"),
        (now + timedelta(hours=1)).strftime("%Y%m%d%H"),
    ]
    channel_ids = [f"bio-{chip}-{stamp}" for (chip,) in rows for stamp in stamps]

    return await mint_token(user_id=user.name, channel_ids=channel_ids)
```

Reuse whatever dependency already authenticates the existing session JWT
(`Backend/app/utils/JWT_Auth.py`) for the `user=Depends(...)` slot.

### 5.4 Frontend

```bash
cd esp-monitor
npm install @portalsdk/core@0.1.5 @portalsdk/react@0.1.4
```

Pin exact versions. Portal is pre-1.0; a `^` range will pull a breaking minor.

```jsx
// esp-monitor/src/App.jsx
import { Portal } from "@portalsdk/core";
import { PortalProvider } from "@portalsdk/react";

const portal = new Portal({ apiKey: import.meta.env.VITE_PORTAL_KEY });

// A callback, not a string: it is re-invoked on connect, reconnect and expiry.
// A plain string cannot refresh and dies with TokenExpiredError after the TTL.
const getToken = async () => {
  const response = await fetch("/api/portal/token", {
    headers: { Authorization: `Bearer ${sessionJwt}` },
  });
  const { token } = await response.json();
  return token;
};

export function App() {
  return (
    <PortalProvider client={portal} token={getToken}>
      {/* ... */}
    </PortalProvider>
  );
}
```

```jsx
// esp-monitor/src/components/Dashboard.jsx
import { useChannel } from "@portalsdk/react";

function channelId(chipId) {
  const now = new Date();
  const stamp = [
    now.getUTCFullYear(),
    String(now.getUTCMonth() + 1).padStart(2, "0"),
    String(now.getUTCDate()).padStart(2, "0"),
    String(now.getUTCHours()).padStart(2, "0"),
  ].join("");
  return `bio-${chipId}-${stamp}`;
}

export function Dashboard({ chipId }) {
  const { messages, status } = useChannel({
    channelId: channelId(chipId),
    history: 1, // last message arrives before first render — never blank
  });

  const latest = messages[messages.length - 1];
  // ...
}
```

`VITE_PORTAL_KEY` holds the `pk_` publishable key. That one is browser-safe by design and
belongs in the bundle. The `sk_` never leaves the backend.

---

## 6. Known Portal limitations

Observed on a separate CloudForge AI project (a realtime tracking demo) against
`@portalsdk/core` 0.1.5 / `@portalsdk/react` 0.1.4, on 2026-08-05/06. Reported to the
Portal team on 2026-08-07. Verified there, **not** re-verified against this project's keys —
re-check before designing around either.

These are not trivia. Two of them are the direct reason §4.1 and §4.2 exist.

### 6.1 Every client→server socket frame is accepted and silently dropped

`channel.send({ ephemeral: true })`, `channel.sendActivity()` / `sendTyping()`, and
`channel.setMetadata()` all resolve without error and never arrive at any other client on
the channel. Persistent `send()` on the same channel, in the same session, works.

Those three are exactly the frames that ride the WebSocket upstream; persistent publishes
go over HTTP instead. The failing set and the socket-upstream set are the same set.

Measured: 5 consecutive ephemeral sends, 400 ms apart — **0 of 5** delivered. No `error`
frame either. Ruled out: plan/quota limits (neither `channel_at_capacity` nor `rate_limit`
fired), identity collision (distinct `channel.me.id` per tab), and channel middleware
(default config, nothing deployed).

**What it means here.** Ephemeral messages carry `seq: null`, are not persisted, and do not
count against channel depth — which makes them the obvious answer for 2-second telemetry.
That answer is unavailable. Hence the throttle in §4.1 and the rotation in §4.2. If this is
fixed upstream, telemetry can move to `ephemeral: true` and both mitigations become
optional.

Also: do not design anything around presence metadata or typing/activity indicators.

### 6.2 Live delivery stops permanently once a channel accumulates history

On a channel that had accumulated on the order of a couple hundred persistent messages, a
client stopped receiving entirely. Observed state at the stall:

```
observer: channel.messages.length = 28    (frozen)
observer: channel.status          = "ready"
observer: channel.unread          = 86    (climbing)
publisher: send() → {id: "m_3_201"}, {id: "m_3_202"}, {id: "m_3_203"}   ← never arrived
```

`unread: 86` next to 28 materialized messages says the client knows it is behind. Nothing
in the public API surface reflects that. Changing only the channel id — same code, same
tabs — restored delivery immediately.

Forward pagination (`loadNext`) is a reserved surface in v1; only `loadPrevious()` exists.
So a client that falls behind cannot catch up without tearing down the channel.

**What it means here.** Two concrete requirements, both already in this plan:

1. Rotate channels (§4.2) so depth never accumulates.
2. Render a staleness indicator (§4.3). A stalled channel and an idle device look
   identical from the client, so the UI must distinguish "nothing has changed" from "we
   have heard nothing in 4 minutes".

### 6.3 Surfaces rejected in v1

Do not plan around these; they throw `NotYetSupportedError`:

- attachments and media
- read receipts
- server-side `where` filtering (client-side `channel.view()` works)
- forward pagination — only `loadPrevious()`
- notifications deliver to the Portal inbox only; no device push, Slack, or email

### 6.4 Other operational facts

- `content` is capped at **2 KB**. Current readings are far under; keep it in mind if
  payloads grow.
- Channel publish has **no idempotency key** (only `POST /v1/users/{id}/notifications`
  does). Any retry logic in the bridge must dedupe on its own, or it will double-publish
  and burn budget.
- The marketing site at useportal.co advertises an API the shipped SDK does not have
  (`channel.publish()`, `channel.subscribe(cb)`, multi-destination `notify`, reactions).
  The Portal team confirmed on 2026-08-07 that those snippets are illustrative only. Treat
  <https://docs.useportal.co> as the only source of truth.

---

## 7. Findings in the current codebase

Independent of the migration. Listed because whoever does this work will be in these files
anyway. Each was verified in the tree.

### 7.1 Committed Wi-Fi credentials

`Esp32/station_main/station_main.ino:10-11` contains a real SSID and password in a public
repository. **If that network is still in use, rotate the password.** Removing the lines in
a later commit does not help — the value stays in git history. Going forward, keep them in
a gitignored `secrets.h`.

### 7.2 The motor task is created twice

`xTaskCreate(controlMotor, ...)` runs at `Esp32/station_main/station_main.ino:372`
(priority 3) and again at `:407` (priority 1). Two FreeRTOS tasks drive the same stepper
through the same globals (`motorStartRequest`, `currentPosition`). The second call should
be removed.

### 7.3 `WebSocketState` is used but never imported

`Backend/app/routes/esp_socket.py:252` references `WebSocketState.CONNECTED` inside the
frontend socket's exception handler. The name is not imported anywhere in the module, so
the handler raises `NameError` precisely when it is needed. Moot once `/ws/frontend` is
deleted, but worth knowing if that code is kept.

### 7.4 Device sockets are unauthenticated

`/ws/esp/{device_id}` verifies only that the device id is *associated with a user*
(`Backend/app/routes/esp_socket.py:61`), never that the connecting client is that device.
Anyone who learns a chip id can connect and stream fabricated readings. Registration is
likewise anonymous, with a hardcoded `int userId = 4` at
`Esp32/station_main/station_main.ino:387`.

This is out of scope for the migration — the LAN boundary is the current control — but it
is the single largest gap if the system ever leaves the local network. §9 notes the path
that closes it.

### 7.5 CORS

`Backend/main.py:74-82` sets `allow_origins=["*"]` together with `allow_credentials=True`.
Browsers reject that combination outright. Once the frontend talks to Portal directly, the
only cross-origin calls left are to FastAPI's own routes, so this should be narrowed to the
actual dashboard origin.

---

## 8. Setup and verification

### Environment

```bash
# Backend/.env
PORTAL_SECRET_KEY=sk_...        # server only. The API rejects any sk_ request
                                # carrying an Origin header, so a browser leak
                                # fails loudly — but do not rely on that as the control.

# esp-monitor/.env
VITE_PORTAL_KEY=pk_...          # publishable, browser-safe, belongs in the bundle
```

Both go in `.gitignore` (`.env` is already listed).

### Verification checklist

Two browsers side by side is the real test — one flying, one watching.

- [ ] Device connects; `SENSOR_DATA` arrives at FastAPI (existing logs).
- [ ] Dashboard renders a reading within a second of it being published.
- [ ] **Close the dashboard tab, wait 30 s, reopen it.** It must render current state
      immediately, before any new message arrives. This is what `history: 1` buys.
- [ ] `START_MOTOR` from the dashboard moves the motor, and the dashboard flips to
      "running" *from the device's echo*, not optimistically.
- [ ] `STOP_MOTOR` stops it. Measure the latency; it should match today's.
- [ ] Let it run 10 minutes and count published messages. Expect roughly 10, not ~300. If
      it is ~300, the throttle is not wired.
- [ ] Log in as a user who does **not** own the device and confirm the channel is refused.
- [ ] Restart FastAPI while the dashboard is open. The dashboard should recover on its own
      once the device reconnects.
- [ ] Run `uvicorn --workers 2` and repeat the motor test. It should now work regardless of
      which worker serves the request — this is the §2.1 fix, and it is worth demonstrating.

---

## 9. Out of scope

Recorded so the next person does not have to rediscover the reasoning.

**Channel extension for durable state.** A Portal extension with `transport: "http"` and an
`onSnapshot()` handler would keep current device state inside Portal, surviving a FastAPI
restart — the durable version of §4.5. Note that `transport: "http"` matters: it routes
namespaced sends as HTTP publishes rather than the ephemeral socket frames that are broken
per §6.1. One thing to verify first: whether a raw HTTP publish with a dotted `type`
reaches the extension's `onBatch` when the send does not come from the SDK. The docs
describe that routing from the client side only.

**AWS IoT Core instead of the LAN WebSocket.** If this ever leaves the local network, the
device leg should be MQTT over mutual TLS with a per-device X.509 certificate, not a raw
WebSocket. That closes §7.4 by construction, and brings QoS 1, Last Will and Testament for
crash detection, persistent sessions, a 128 KB payload ceiling, and an 8 KB Device Shadow
with `desired`/`reported`/`delta` reconciliation. FastAPI's bridge role would move to a
Lambda behind an IoT rule, and the rule's SQL becomes the place where you decide what is
worth forwarding to Portal. Deliberately excluded here: the current goal is a working demo,
and that migration is independent of this one.

**Do both migrations separately.** Portal first, against the FastAPI that exists today, so
the realtime layer can be validated on its own. Any device-transport change comes after.

---

## References

- Portal documentation — <https://docs.useportal.co>
- `wire-protocol` (frames, ordering, gap-fill, reserved surfaces) —
  <https://docs.useportal.co/wire-protocol>
- SSR & Next.js (the hooks are SSR-inert; `dynamic({ ssr: false })` is not required) —
  <https://docs.useportal.co/core/ssr-and-nextjs>
- AWS IoT Core quotas (§9) —
  <https://docs.aws.amazon.com/general/latest/gr/iot-core.html>
