# Smart Home System — BTN415 Project (Group 6)

A simulated smart home that you control from a web dashboard. The core of the project is a **multi-threaded C++ TCP server** that owns the state of every smart device. Clients talk to it over raw TCP sockets using a small text protocol modelled on HTTP. A Node.js bridge translates the browser's HTTP/JSON requests into that TCP protocol, so the React frontend can drive the same server that the C++ command-line client uses.

```
 ┌──────────────┐  HTTP/JSON   ┌──────────────┐  HTTP/JSON  ┌──────────────┐   TCP text   ┌───────────────────┐
 │   Browser    │ ───────────► │  Vite dev    │ ──────────► │ Node bridge  │ ───────────► │  C++ TCP server   │
 │ React UI     │ ◄─────────── │  server      │ ◄────────── │ (Express)    │ ◄─────────── │  + DeviceManager  │
 └──────────────┘  :5173       │ proxy /api   │   :5001     └──────────────┘    :8080     └───────────────────┘
                               └──────────────┘                                                    ▲
                                                                       ┌──────────────┐   TCP text │
                                                                       │ C++ CLI      │ ───────────┘
                                                                       │ client_app   │
                                                                       └──────────────┘
```

---

## Project layout

| Path | What it is |
|---|---|
| `backend/server/` | C++ TCP server (`TCPServer`, `ClientHandler`), entry point `main_server.cpp` |
| `backend/client/` | C++ interactive TCP client (`TCPClient`), entry point `main_client.cpp` |
| `backend/devices/` | Simulated devices (`Light`, `Thermostat`, `SecurityCamera`) and the `DeviceManager` |
| `backend/utilities/` | Cross-platform socket helpers (`SocketUtils.h`, `SocketSystem`) and `CommandParser.h` |
| `backend/bridge/` | Node.js/Express HTTP → TCP bridge (`server.js`) |
| `backend/tests/` | Unit tests for the command parser |
| `backend/demos/` | Stand-alone demo of subnetting, routing and ARP decisions |
| `frontend/` | React + Vite + Tailwind dashboard (one page per room) |

---

## How to run

You need three terminals. Start them in this order.

**1. C++ server (port 8080)**
```bash
cd backend
g++ -std=c++17 -pthread server/main_server.cpp server/Server.cpp utilities/SocketSystem.cpp devices/DeviceManager.cpp -o server/server_app
./server/server_app
```

**2. Node bridge (port 5001)**
```bash
cd backend/bridge
npm install
npm start
```

**3. React frontend (port 5173)**
```bash
cd frontend
npm install
npm run dev
```

Open http://localhost:5173.

### Optional: C++ command-line client
```bash
cd backend
g++ -std=c++17 -pthread client/main_client.cpp client/Client.cpp utilities/SocketSystem.cpp -o client/client_app
./client/client_app
> GET /light/on
Server: 200 OK: living_room_light turned ON
```

You can also talk to the server with no client at all:
```bash
printf "GET /all/status\n" | nc 127.0.0.1 8080
```

### Tests and network demo
```bash
cd backend
g++ -std=c++17 tests/test_command_parser.cpp -o tests/test_command_parser && ./tests/test_command_parser
g++ -std=c++17 demos/network_routing_demo.cpp -o demos/network_routing_demo && ./demos/network_routing_demo
```

On Windows the same sources build with MSVC/MinGW. `SocketSystem` calls `WSAStartup`/`WSACleanup`, and you link `ws2_32`.

---

## Data network communication — how it works

### 1. Transport: TCP sockets (Berkeley / Winsock API)

The server and the CLI client use the OS socket API directly, with no networking library.

**Server side** (`backend/server/Server.cpp`):

| Step | Call | Meaning |
|---|---|---|
| Create | `socket(AF_INET, SOCK_STREAM, 0)` | IPv4 (`AF_INET`) + TCP (`SOCK_STREAM`) |
| Bind | `bind()` with `sin_port = htons(8080)`, `sin_addr = INADDR_ANY` | Claim port 8080 on every network interface. `htons` converts the port to **network byte order** (big-endian). |
| Listen | `listen(sock, SOMAXCONN)` | Mark the socket passive and let the kernel queue incoming connections |
| Accept | `accept()` (blocking loop) | The kernel completes the TCP 3-way handshake (SYN → SYN-ACK → ACK) and returns a **new socket** for that one client |
| Receive / send | `recv()` / `send()` | Read a command, write a response |
| Close | `close()` / `closesocket()` | Sends FIN, which ends the connection |

**Client side** (`backend/client/Client.cpp`): `socket()` → `inet_pton("127.0.0.1")` converts the dotted-decimal string to a binary address → `connect()` to port 8080. After that, each typed line is sent with `send()` and the reply is read with `recv()`.

TCP gives the project **reliable, ordered, connection-oriented** delivery. Lost packets are retransmitted and bytes arrive in order, so the application never has to handle packet loss itself.

**Portability:** `SocketUtils.h` hides the differences between POSIX sockets and Winsock (`SOCKET` vs `int`, `closesocket` vs `close`, `WSAGetLastError` vs `errno`). `SocketSystem` is an RAII wrapper that starts and stops Winsock on Windows.

### 2. Concurrency: one thread per client

`TCPServer::accept_clients()` runs in a loop. Every accepted connection gets its own detached `std::thread` running `ClientHandler::run()`, so one slow client never blocks the others. Shared state is protected as follows:

- **`clients` vector + `clients_mutex`:** tracks the active sockets and prints the active-client count on connect and disconnect.
- **`DeviceManager::m_managerMutex`:** serialises access to the device table.
- **`Device::m_mutex`:** each device locks its own state. Each device also runs a background **worker thread** that "ticks" once per second. For example, the thermostat moves its current temperature 1° toward the target, and the camera counts frames while it is recording.

This means the system handles real concurrent access: the browser (through the bridge) and several CLI clients can change the same devices at the same time without race conditions.

### 3. Application-layer protocol (HTTP-style text over TCP)

The project defines its own small text protocol. It is modelled on an HTTP request line and status codes.

**Request:** one line, terminated by `\n`
```
GET /<device>/<action>
```
`CommandParser.h` checks that the line starts with `GET `, strips the leading `/`, and splits the rest at the first `/` into `device` and `action`.

**Response:** a status code followed by a message, terminated by `\n`

| Code | Meaning | Example |
|---|---|---|
| `200 OK` | Success | `200 OK: living_room_light turned ON` |
| `400 ERROR` | Bad action or value | `400 ERROR: temperature must be between 10 and 35` |
| `404 ERROR` | Unknown device | `404 ERROR: device not found` |
| `ERROR` | Malformed request line | `ERROR: invalid command format. Use GET /device/action` |

**Devices and actions the server supports:**

| Device key | Actions |
|---|---|
| `light` | `on`, `off`, `status`, `brightness:<0-100>` |
| `thermostat` | `on`, `off`, `status`, `set:<10-35>` |
| `camera` | `on`, `off`, `status`, `record:on`, `record:off`, `motion:on`, `motion:off` |
| `all` | `status`: returns every device's status, one per line |

Example session:
```
GET /thermostat/on        -> 200 OK: main_thermostat turned ON
GET /thermostat/set:25    -> 200 OK: main_thermostat target temperature set to 25
GET /camera/record:on     -> 400 ERROR: camera must be ON before recording
GET /oven/on              -> 404 ERROR: device not found
```

**Framing:** messages are delimited by newlines. The server reads with a single `recv()` into a 1 KB buffer and strips trailing `\r\n`. Because TCP is a byte stream rather than a message stream, this relies on each command arriving in one segment. That holds for these short commands on a LAN or localhost.

### 4. The bridge: protocol translation (HTTP ↔ TCP)

Browsers can't open raw TCP sockets, so `backend/bridge/server.js` acts as an **application-layer gateway**:

1. Express listens for HTTP on **port 5001**, with CORS enabled so the browser is allowed to call it.
2. The bridge maps each REST route to a protocol command. For example, `POST /api/devices/light/on` becomes `GET /light/on`.
3. For each request it opens a **short-lived TCP connection** to `127.0.0.1:8080` with Node's `net.Socket`. It writes the command plus `\n`, waits for a reply line, then closes the connection. A 5-second timeout guards against a server that has stopped responding.
4. It wraps the reply in JSON (`{ success, message, raw }`) and sends it back to the browser.
5. `GET /api/health` does a **TCP connect probe**: it opens and immediately closes a connection to port 8080 to report whether the C++ server is up.

The two client styles show two connection models:

- **Persistent connection:** the C++ CLI client connects once and sends many requests over the same socket.
- **Per-request connection:** the bridge opens one TCP connection per HTTP request, like HTTP/1.0.

| Bridge route | Sent to C++ server |
|---|---|
| `GET /api/health` | TCP connect probe only |
| `GET /api/devices` | `GET /devices/status` |
| `GET /api/devices/:device` | `GET /:device/status` |
| `POST /api/devices/:device/:action` | `GET /:device/:action[/value][?room=…]` |
| `POST /api/command` `{ "command": "…" }` | the raw command, unchanged |

### 5. Frontend → bridge (HTTP and the Vite proxy)

The React app runs on the Vite dev server (**port 5173**). It reaches the bridge in two ways:

- **Relative URLs through the Vite proxy:** the pages poll `/api/health` every 15 seconds to show whether the bridge and the C++ server are online. `vite.config.js` proxies anything under `/api` to `http://localhost:5001`, so the browser only talks to its own origin.
- **Cross-origin calls:** the device cards use `frontend/src/utils/api.js`, which calls `VITE_API_URL` (set in `frontend/.env` to `http://localhost:5001/api`) directly. Because 5173 → 5001 is a different origin, the browser runs **CORS** checks. That is why the bridge uses the `cors()` middleware.

### 6. Network-layer concepts demo (`backend/demos/network_routing_demo.cpp`)

The project runs over localhost, so the OS handles routing invisibly. This stand-alone program shows the decisions the kernel makes before a packet leaves a host:

- **Subnet membership:** `(src & mask) == (dst & mask)` decides whether a destination is on the local network or remote.
- **Routing:** **longest-prefix match** over a routing table, falling back to the **default gateway** (`0.0.0.0/0`).
- **ARP:** resolves the **next hop's** IP to a MAC address. For a remote destination, the next hop is the gateway, not the final destination.
- **ICMP echo (ping):** shows how the request is encapsulated in IP and then in an Ethernet frame addressed to the next hop's MAC.

### End-to-end walkthrough: clicking "Light ON"

1. React calls `api.controlLight('on', roomId)`, which sends **HTTP POST** `http://localhost:5001/api/devices/light/on` (a cross-origin request, so the browser runs CORS checks).
2. The bridge (Express on :5001) matches the route.
3. The bridge opens a TCP connection to 127.0.0.1:8080 (3-way handshake) and sends `GET /light/on\n`.
4. On the server, `accept()` returns a socket, a new `ClientHandler` thread starts, `recv()` reads the line, `parseCommand` splits it into `light` / `on`, and `DeviceManager` locks the `Light` and sets its power on.
5. The server `send()`s `200 OK: living_room_light turned ON\n`.
6. The bridge reads the line, closes the connection (FIN), and replies with JSON.
7. React updates the card.

---

## Known limitations / mismatches

These are gaps between the frontend/bridge and what the C++ server currently implements. I confirmed them by sending each command to the running server with `nc`.

- **Only three device types exist on the server:** `light`, `thermostat` and `camera`. The UI also has cards for an oven, fridge, shower, mirror, speaker, TV, blinds, garage and exhaust fan. The server answers those with `404 ERROR: device not found`.
- **Rooms aren't modelled.** The bridge appends `?room=<id>`, so the server receives an action like `on?room=kitchen` and replies `400 ERROR: invalid action`. All rooms share the same single light, thermostat and camera.
- **Value syntax differs.** The bridge and CLI help text use `set/22`, but the devices expect `set:22` and `brightness:70`.
- **All-devices status:** the bridge sends `GET /devices/status`, but the server's command is `GET /all/status`. The multi-line reply would also be cut off, because the bridge stops reading at the first `\n`.
- **Status parsing:** the bridge looks for replies starting with `OK:` or `ERROR:`, but the server sends `200 OK:` and `400 ERROR:`. So every reply is reported to the browser as `success: true`.
- **Terminal box:** the room pages send `GET /api/command?cmd=...`, but the bridge only accepts `POST /api/command` with a JSON body.
- **`api.js` fallback URL** is `http://localhost:5000/api`, but the bridge runs on 5001. It works only because `frontend/.env` sets `VITE_API_URL`.
