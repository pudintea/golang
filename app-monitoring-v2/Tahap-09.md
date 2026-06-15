Sip, itu urutan yang benar. API dulu → baru WebSocket. Karena WebSocket nanti cuma “numpang data” dari API/collector yang sudah stabil.

Kita mulai sekarang: **API `/api/summary`**

---

# 1. Tujuan API `/api/summary`

Endpoint ini akan mengembalikan data realtime terbaru:

```json id="k9q2lm"
{
  "cpu_usage": 25.7,
  "cpu_core": 4,

  "ram_total": 17000000000,
  "ram_used": 12000000000,
  "ram_usage": 70,

  "disk_total": 100000000000,
  "disk_used": 48000000000,
  "disk_usage": 48,

  "rx_speed": 12000,
  "tx_speed": 5000,

  "uptime": 123456,
  "load1": 0.5,
  "load5": 0.6,
  "load15": 0.7
}
```

---

# 2. Tambahkan Dependency API Server (Gin)

Pastikan sudah ada:

```bash id="n4kq8x"
go get github.com/gin-gonic/gin
```

---

# 3. Buat API Layer

Buat file:

```text id="a9k2dp"
internal/api/summary.go
```

---

## IMPLEMENTASI `/api/summary`

```go id="p3l8xq"
package api

import (
	"net/http"

	"github.com/gin-gonic/gin"

	"pudin.eu.org/app-monitor/internal/collector"
	"pudin.eu.org/app-monitor/internal/system"
)

func SummaryHandler() gin.HandlerFunc {

	return func(c *gin.Context) {

		cpuUsage, cpuCore, _ := collector.GetCPUUsage()

		ramTotal, ramUsed, ramUsage, _ := collector.GetMemory()

		diskTotal, diskUsed, diskUsage, _ := collector.GetDisk()

		net, _ := collector.GetNetworkSpeed()

		load1, load5, load15 := system.LoadAverage()
		uptime := system.Uptime()

		c.JSON(http.StatusOK, gin.H{
			"cpu_usage": cpuUsage,
			"cpu_core":  cpuCore,

			"ram_total": ramTotal,
			"ram_used":  ramUsed,
			"ram_usage": ramUsage,

			"disk_total": diskTotal,
			"disk_used":  diskUsed,
			"disk_usage": diskUsage,

			"rx_speed": net.RXSpeed,
			"tx_speed": net.TXSpeed,

			"uptime":  uptime,
			"load1":   load1,
			"load5":   load5,
			"load15":  load15,
		})
	}
}
```

---

# 4. Tambahkan System Load + Uptime

Buat file:

```text id="x2m9qv"
internal/system/load.go
```

```go id="q8w3nd"
package system

import (
	"fmt"
	"os"
	"strconv"
	"strings"
)

func LoadAverage() (float64, float64, float64) {

	data, err := os.ReadFile("/proc/loadavg")
	if err != nil {
		return 0, 0, 0
	}

	parts := strings.Fields(string(data))

	load1, _ := strconv.ParseFloat(parts[0], 64)
	load5, _ := strconv.ParseFloat(parts[1], 64)
	load15, _ := strconv.ParseFloat(parts[2], 64)

	return load1, load5, load15
}

func Uptime() int64 {

	data, err := os.ReadFile("/proc/uptime")
	if err != nil {
		return 0
	}

	parts := strings.Fields(string(data))

	val, _ := strconv.ParseFloat(parts[0], 64)

	return int64(val)
}

func FormatUptime(seconds int64) string {

	min := seconds / 60
	hour := min / 60
	day := hour / 24

	return fmt.Sprintf("%d days %d hours %d minutes",
		day,
		hour%24,
		min%60,
	)
}
```

---

# 5. Update Main (RUN API SERVER)

Sekarang kita ubah `main.go` supaya API hidup.

```go id="v7k2xq"
package main

import (
	"log"

	"github.com/gin-gonic/gin"

	"pudin.eu.org/app-monitor/internal/api"
	"pudin.eu.org/app-monitor/internal/database"
	"pudin.eu.org/app-monitor/internal/scheduler"
	"pudin.eu.org/app-monitor/internal/system"
)

func main() {

	db, err := database.NewSQLite()
	if err != nil {
		log.Fatal(err)
	}

	err = database.Migrate(db)
	if err != nil {
		log.Fatal(err)
	}

	_ = database.SaveServerInfo(
		db,
		system.Hostname(),
		system.OSVersion(),
		system.Kernel(),
		system.Arch(),
	)

	// jalanin collector di background
	go scheduler.StartCollector(db)

	// setup gin
	r := gin.Default()

	r.GET("/api/summary", api.SummaryHandler())

	log.Println("Server running on :8087")
	r.Run(":8087")
}
```

---

# 6. TEST API

Jalankan:

```bash id="w1q9lm"
go run ./cmd/app-monitor
```

---

## Test endpoint:

```bash id="c8n2xv"
curl http://localhost:8087/api/summary
```

---

## Hasil harus seperti ini:

```json id="z3k8qp"
{
  "cpu_usage": 12.3,
  "cpu_core": 4,
  "ram_total": 17000000000,
  "ram_used": 12000000000,
  "ram_usage": 70,
  "disk_total": 100000000000,
  "disk_used": 48000000000,
  "disk_usage": 48,
  "rx_speed": 12000,
  "tx_speed": 5000,
  "uptime": 123456,
  "load1": 0.2,
  "load5": 0.3,
  "load15": 0.4
}
```

---

# 7. Kalau sudah berhasil

Kita sudah punya:

```text id="k3m8qp"
✔ Collector
✔ SQLite logging
✔ Network speed
✔ System info
✔ API summary (REALTIME)
```

---

# NEXT STEP (SETELAH INI)

Kalau API ini sudah jalan, kita lanjut ke:

## 👉 WebSocket Realtime Engine (/ws)

Yang akan:

* push data tiap 10 detik
* update dashboard tanpa refresh
* jadi dasar Chart.js live

---

Kalau kamu sudah berhasil test `/api/summary`, bilang saja:

👉 “API sudah jalan”

nanti kita langsung lanjut WebSocket engine.
