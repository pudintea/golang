Mantap. Dengan spesifikasi yang sudah terkunci, saya sarankan kita mulai dari desain arsitektur yang benar terlebih dahulu, agar nanti tidak perlu refactor besar saat aplikasi berkembang.

# Arsitektur Final Aplikasi Monitor

## Teknologi

### Backend

* Golang 1.24+
* SQLite
* WebSocket
* REST API

Library yang saya rekomendasikan:

```bash
go get github.com/shirou/gopsutil/v4
go get github.com/gorilla/websocket
go get github.com/mattn/go-sqlite3
go get github.com/gin-gonic/gin
```

Alasan:

* gopsutil → ambil CPU, RAM, SSD, Network, Uptime
* websocket → realtime dashboard
* sqlite3 → database
* gin → HTTP server ringan dan cepat

---

# Struktur Folder

```text
app-monitor/
│
├── cmd/
│   └── app-monitor/
│       └── main.go
│
├── internal/
│   │
│   ├── collector/
│   │   ├── cpu.go
│   │   ├── memory.go
│   │   ├── disk.go
│   │   ├── network.go
│   │   ├── uptime.go
│   │   └── loadavg.go
│   │
│   ├── database/
│   │   ├── sqlite.go
│   │   └── migration.go
│   │
│   ├── models/
│   │   └── monitor.go
│   │
│   ├── scheduler/
│   │   ├── collector.go
│   │   └── cleanup.go
│   │
│   ├── websocket/
│   │   └── hub.go
│   │
│   ├── api/
│   │   ├── dashboard.go
│   │   ├── history.go
│   │   └── websocket.go
│   │
│   └── system/
│       ├── hostname.go
│       ├── os.go
│       └── kernel.go
│
├── web/
│   ├── templates/
│   │   └── index.html
│   │
│   ├── static/
│   │   ├── css/
│   │   ├── js/
│   │   └── img/
│
├── data/
│   └── monitor.db
│
├── configs/
│   └── config.yaml
│
└── build/
```

---

# Struktur Database

Tabel tunggal saja.

```sql
CREATE TABLE monitor_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,

    created_at DATETIME NOT NULL,

    cpu_usage REAL,
    cpu_core INTEGER,

    ram_total INTEGER,
    ram_used INTEGER,
    ram_usage REAL,

    disk_total INTEGER,
    disk_used INTEGER,
    disk_usage REAL,

    rx_bytes INTEGER,
    tx_bytes INTEGER,

    rx_speed REAL,
    tx_speed REAL
);
```

Index:

```sql
CREATE INDEX idx_created_at
ON monitor_logs(created_at);
```

---

# Interval Monitoring

Setiap:

```text
10 detik
```

Scheduler:

```go
ticker := time.NewTicker(10 * time.Second)
```

Alur:

```text
Collect Data
      ↓
Insert SQLite
      ↓
Broadcast WebSocket
```

Walaupun browser tidak dibuka:

```text
SQLite tetap menerima data.
```

---

# Endpoint API

## Dashboard

```http
GET /
```

Render HTML.

---

## Summary

```http
GET /api/summary
```

Response:

```json
{
  "cpu_core": 2,
  "cpu_usage": 37.5,

  "ram_total": 8589934592,
  "ram_usage": 31.2,

  "disk_total": 42949672960,
  "disk_usage": 29.8,

  "uptime": "15 days",

  "load1": 0.45,
  "load5": 0.62,
  "load15": 0.78
}
```

---

## History

```http
GET /api/history?hours=1
```

```http
GET /api/history?hours=6
```

```http
GET /api/history?hours=24
```

---

## WebSocket

```http
/ws
```

Realtime push:

```json
{
  "cpu_usage": 37,
  "ram_usage": 30,
  "disk_usage": 40,
  "rx_speed": 124000,
  "tx_speed": 44000
}
```

---

# Landing Page

## Header

```text
Aplikasi Monitor
pudin.eu.org
```

---

## Top Cards

### CPU

Circular Gauge

```text
37%
2 Core
```

---

### RAM

Circular Gauge

```text
30%
8 GB
```

---

### SSD

Circular Gauge

```text
30%
40 GB
```

---

### Network

```text
RX 1.2 MB/s
TX 0.4 MB/s
```

---

# Server Information

```text
Hostname
OS
Kernel
Architecture
Uptime
Load Average
```

Contoh:

```text
Hostname    : monitor-01
OS          : Ubuntu 24.04
Kernel      : 6.8.0
Arch        : amd64
Uptime      : 15 Days
Load Avg    : 0.12 / 0.32 / 0.41
```

---

# Tema Dashboard

Saya membayangkan seperti ini:

```text
Background:
#0f172a

Card:
#1e293b

Border:
#334155

Accent:
#3b82f6

Success:
#22c55e

Warning:
#f59e0b

Danger:
#ef4444
```

Style:

* Modern SaaS
* Dark Mode default
* Responsive Mobile
* Rounded-xl
* Shadow-lg
* Smooth Animation

---

# Retensi Data

Setiap 1 jam:

```sql
DELETE FROM monitor_logs
WHERE created_at < datetime('now','-24 hours');
```

Database akan selalu kecil.

Perhitungan:

```text
8640 record / hari
```

SQLite sangat mampu menangani ini.

---

# File Binary

Hasil build:

```bash
go build -ldflags="-s -w" \
-o app-monitor \
./cmd/app-monitor
```

Ukuran biasanya:

```text
10–25 MB
```

---

# Service Ubuntu

```ini
[Unit]
Description=Aplikasi Monitor
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/app-monitor
ExecStart=/opt/app-monitor/app-monitor
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

### Satu fitur tambahan yang menurut saya sangat layak masuk versi 1.0

Tambahkan **export CSV**.

Tombol:

```text
Export Data
```

Misalnya:

```http
GET /api/export?hours=24
```

Menghasilkan:

```csv
time,cpu,ram,disk,rx,tx
...
```

Saat ada lonjakan CPU atau jaringan, Anda bisa langsung mengunduh data dan menganalisisnya di Excel atau LibreOffice tanpa perlu membuka database SQLite. Ini fitur kecil tetapi sangat berguna.
