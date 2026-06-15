Gas, ini bagian yang bikin aplikasi kamu naik kelas: **real-time engine** 🔥

Kita bangun WebSocket yang:

* bisa jalan di HTTP (dev)
* siap HTTPS (production)
* broadcast data monitoring ke semua client
* siap dipakai Chart.js realtime

---

# 1. Konsep WebSocket Kita

Struktur alur:

```text id="ws1"
Collector (10 detik)
        ↓
     SQLite
        ↓
   WebSocket Hub
        ↓
  Semua Browser Client
```

Jadi dashboard tidak query terus ke API, tapi **ditarik (push)** dari server.

---

# 2. Install Dependency

```bash id="ws2"
go get github.com/gorilla/websocket
```

---

# 3. WebSocket Hub (CORE ENGINE)

Buat file:

```text id="ws3"
internal/websocket/hub.go
```

---

## IMPLEMENTASI HUB

```go id="ws4"
package websocket

import (
	"log"
	"net/http"
	"sync"

	"github.com/gorilla/websocket"
)

type Client struct {
	Conn *websocket.Conn
	Send chan []byte
}

type Hub struct {
	Clients    map[*Client]bool
	Broadcast  chan []byte
	Register   chan *Client
	Unregister chan *Client
	mu         sync.Mutex
}

func NewHub() *Hub {
	return &Hub{
		Clients:    make(map[*Client]bool),
		Broadcast:  make(chan []byte),
		Register:   make(chan *Client),
		Unregister: make(chan *Client),
	}
}

func (h *Hub) Run() {

	for {
		select {

		case client := <-h.Register:
			h.mu.Lock()
			h.Clients[client] = true
			h.mu.Unlock()
			log.Println("client connected")

		case client := <-h.Unregister:
			h.mu.Lock()
			if _, ok := h.Clients[client]; ok {
				delete(h.Clients, client)
				close(client.Send)
			}
			h.mu.Unlock()
			log.Println("client disconnected")

		case message := <-h.Broadcast:
			h.mu.Lock()
			for client := range h.Clients {
				select {
				case client.Send <- message:
				default:
					close(client.Send)
					delete(h.Clients, client)
				}
			}
			h.mu.Unlock()
		}
	}
}
```

---

# 4. WebSocket Handler

Buat:

```text id="ws5"
internal/api/ws.go
```

---

## HANDLER

```go id="ws6"
package api

import (
	"net/http"

	"github.com/gorilla/websocket"

	ws "pudin.eu.org/app-monitor/internal/websocket"
)

var upgrader = websocket.Upgrader{
	CheckOrigin: func(r *http.Request) bool {
		return true // penting untuk dev + production proxy
	},
}

func WSHandler(hub *ws.Hub) http.HandlerFunc {

	return func(w http.ResponseWriter, r *http.Request) {

		conn, err := upgrader.Upgrade(w, r, nil)
		if err != nil {
			return
		}

		client := &ws.Client{
			Conn: conn,
			Send: make(chan []byte),
		}

		hub.Register <- client

		go func() {
			defer func() {
				hub.Unregister <- client
				conn.Close()
			}()

			for {
				_, _, err := conn.ReadMessage()
				if err != nil {
					break
				}
			}
		}()

		go func() {
			for msg := range client.Send {
				_ = conn.WriteMessage(websocket.TextMessage, msg)
			}
		}()
	}
}
```

---

# 5. Integrasi ke Main

Sekarang kita hubungkan semuanya.

Edit:

```text id="ws7"
cmd/app-monitor/main.go
```

---

## VERSION FINAL MAIN + WS

```go id="ws8"
package main

import (
	"encoding/json"
	"log"
	"time"

	"github.com/gin-gonic/gin"

	"pudin.eu.org/app-monitor/internal/api"
	"pudin.eu.org/app-monitor/internal/database"
	"pudin.eu.org/app-monitor/internal/scheduler"
	"pudin.eu.org/app-monitor/internal/system"
	ws "pudin.eu.org/app-monitor/internal/websocket"
)

func main() {

	db, err := database.NewSQLite()
	if err != nil {
		log.Fatal(err)
	}

	_ = database.Migrate(db)

	_ = database.SaveServerInfo(
		db,
		system.Hostname(),
		system.OSVersion(),
		system.Kernel(),
		system.Arch(),
	)

	// HUB WEBSOCKET
	hub := ws.NewHub()
	go hub.Run()

	// COLLECTOR (background)
	go scheduler.StartCollector(db)

	// SIMULASI PUSH DATA KE WS
	go func() {

		for {

			data := gin.H{
				"type": "realtime",
				"time": time.Now().Unix(),
			}

			b, _ := json.Marshal(data)
			hub.Broadcast <- b

			time.Sleep(10 * time.Second)
		}
	}()

	r := gin.Default()

	r.GET("/api/summary", api.SummaryHandler())

	r.GET("/ws", api.WSHandler(hub))

	log.Println("Server running :8087")
	r.Run(":8087")
}
```

---

# 6. TEST WEBSOCKET

## Browser Console:

```javascript id="ws9"
let ws = new WebSocket("ws://localhost:8087/ws");

ws.onmessage = (msg) => {
    console.log("DATA:", msg.data);
};
```

Harus muncul:

```text id="ws10"
DATA: {"type":"realtime","time":171844xxxx}
```

setiap 10 detik.

---

# 7. HTTPS SUPPORT (IMPORTANT)

Di production Ubuntu nanti:

## Option A (RECOMMENDED): Nginx Reverse Proxy

```text id="ws11"
HTTPS → Nginx → http://localhost:8087
```

WebSocket otomatis support:

```text id="ws12"
wss://domain.com/ws
```

---

## Option B (native Go TLS)

Nanti bisa pakai:

```go id="ws13"
http.ListenAndServeTLS(":443", "cert.pem", "key.pem", r)
```

Tapi untuk sekarang:
👉 kita tetap pakai Nginx (lebih profesional)

---

# STATUS SEKARANG

Kamu sudah sampai sini:

```text id="ws14"
████████████████░░░░ 75–80%
```

✔ Collector
✔ SQLite
✔ API
✔ WebSocket Hub
✔ Real-time push
✔ Ready for dashboard

---

# NEXT STEP (FINAL BIG STEP)

Sekarang tinggal:

# 👉 DASHBOARD UI (Tailwind + Chart.js REALTIME)

Akan kita bangun:

* CPU gauge (circle)
* RAM gauge
* SSD gauge
* Network live graph
* Load average panel
* Uptime panel
* LIVE update via WebSocket

---

Kalau kamu siap, bilang:

👉 **“lanjut dashboard UI”**

dan kita bikin tampilan yang mulai terlihat seperti **Netdata / Grafana mini versi kamu sendiri** 🔥
