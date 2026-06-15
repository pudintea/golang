# Update main.go jadi begini kalau tidak bisa satart
Kode menjalankan aplikasi
```
go run ./cmd/app-monitor
```
Ini Kode main.go
```
package main

import (
	"encoding/json"
	"log"
	"time"

	"github.com/gin-gonic/gin"

	"pudin.eu.org/app-monitor/internal/api"
	"pudin.eu.org/app-monitor/internal/collector"
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

	// ===== WEBSOCKET HUB =====
	hub := ws.NewHub()
	go hub.Run()

	// ===== COLLECTOR (BACKGROUND) =====
	go scheduler.StartCollector(db)

	// ===== REALTIME PUSH =====
	go func() {

		for {

			summary := gin.H{}

			// ambil data real
			cpuUsage, cpuCore, _ := collector.GetCPUUsage()
			ramTotal, ramUsed, ramUsage, _ := collector.GetMemory()
			diskTotal, diskUsed, diskUsage, _ := collector.GetDisk()
			net, _ := collector.GetNetworkSpeed()
			load1, load5, load15 := system.LoadAverage()
			uptime := system.Uptime()

			summary = gin.H{
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

				"load1":  load1,
				"load5":  load5,
				"load15": load15,

				"uptime": uptime,
			}

			b, _ := json.Marshal(summary)
			hub.Broadcast <- b

			time.Sleep(10 * time.Second)
		}
	}()

	// ===== GIN SERVER =====
	gin.SetMode(gin.ReleaseMode)

	r := gin.Default()

	r.Static("/static", "./web/static")
	r.LoadHTMLGlob("web/templates/*")

	r.GET("/", func(c *gin.Context) {
		c.HTML(200, "index.html", nil)
	})

	r.GET("/api/summary", api.SummaryHandler())
	r.GET("/ws", api.WSHandler(hub))

	log.Println("Server running on :8087")

	// INI WAJIB DI AKHIR (BLOCKING)
	err = r.Run(":8087")
	if err != nil {
		log.Fatal(err)
	}
}

```
## Pudin Saepudin
