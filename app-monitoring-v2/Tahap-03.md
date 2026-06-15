Siap. Sebelum menulis ribuan baris kode, saya sarankan kita mulai dengan fondasi yang benar agar proyek mudah dikembangkan.

## Tahap 1: Inisialisasi Project

Struktur awal:

```text
app-monitor/
├── cmd/
│   └── app-monitor/
│       └── main.go
├── internal/
│   ├── collector/
│   ├── database/
│   ├── models/
│   ├── scheduler/
│   ├── api/
│   ├── websocket/
│   └── system/
├── web/
│   ├── templates/
│   └── static/
├── data/
├── go.mod
└── go.sum
```

Buat project:

```bash
mkdir app-monitor
cd app-monitor

go mod init pudin.eu.org/app-monitor
```

Install dependency:

```bash
go get github.com/gin-gonic/gin
go get github.com/gorilla/websocket
go get github.com/mattn/go-sqlite3
go get github.com/shirou/gopsutil/v4
```

---

# Tahap 2: Model Database

Buat file:

```text
internal/models/monitor.go
```

```go
package models

import "time"

type MonitorLog struct {
	ID        int64     `db:"id"`
	CreatedAt time.Time `db:"created_at"`

	CPUUsage float64 `db:"cpu_usage"`
	CPUCore  int     `db:"cpu_core"`

	RAMTotal uint64  `db:"ram_total"`
	RAMUsed  uint64  `db:"ram_used"`
	RAMUsage float64 `db:"ram_usage"`

	DiskTotal uint64  `db:"disk_total"`
	DiskUsed  uint64  `db:"disk_used"`
	DiskUsage float64 `db:"disk_usage"`

	RXBytes uint64 `db:"rx_bytes"`
	TXBytes uint64 `db:"tx_bytes"`

	RXSpeed float64 `db:"rx_speed"`
	TXSpeed float64 `db:"tx_speed"`
}
```

---

# Tahap 3: SQLite

Buat file:

```text
internal/database/sqlite.go
```

```go
package database

import (
	"database/sql"

	_ "github.com/mattn/go-sqlite3"
)

func NewSQLite() (*sql.DB, error) {
	return sql.Open("sqlite3", "./data/monitor.db")
}
```

---

Buat migration:

```text
internal/database/migration.go
```

```go
package database

import "database/sql"

func Migrate(db *sql.DB) error {

	query := `
	CREATE TABLE IF NOT EXISTS monitor_logs (
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

	CREATE INDEX IF NOT EXISTS idx_created_at
	ON monitor_logs(created_at);
	`

	_, err := db.Exec(query)
	return err
}
```

---

# Tahap 4: CPU Collector

File:

```text
internal/collector/cpu.go
```

```go
package collector

import (
	"runtime"

	"github.com/shirou/gopsutil/v4/cpu"
)

func GetCPUUsage() (float64, int, error) {

	percent, err := cpu.Percent(0, false)
	if err != nil {
		return 0, 0, err
	}

	return percent[0], runtime.NumCPU(), nil
}
```

---

# Tahap 5: RAM Collector

File:

```text
internal/collector/memory.go
```

```go
package collector

import "github.com/shirou/gopsutil/v4/mem"

func GetMemory() (uint64, uint64, float64, error) {

	vm, err := mem.VirtualMemory()
	if err != nil {
		return 0, 0, 0, err
	}

	return vm.Total, vm.Used, vm.UsedPercent, nil
}
```

---

# Tahap 6: SSD Collector

File:

```text
internal/collector/disk.go
```

```go
package collector

import "github.com/shirou/gopsutil/v4/disk"

func GetDisk() (uint64, uint64, float64, error) {

	usage, err := disk.Usage("/")
	if err != nil {
		return 0, 0, 0, err
	}

	return usage.Total,
		usage.Used,
		usage.UsedPercent,
		nil
}
```

---

# Tahap 7: Network Collector

File:

```text
internal/collector/network.go
```

Versi awal:

```go
package collector

import "github.com/shirou/gopsutil/v4/net"

func GetNetwork() (uint64, uint64, error) {

	stats, err := net.IOCounters(false)
	if err != nil {
		return 0, 0, err
	}

	return stats[0].BytesRecv,
		stats[0].BytesSent,
		nil
}
```

Nanti kita upgrade supaya otomatis mendeteksi interface utama (`eth0`, `ens3`, dll).

---

# Tahap 8: Main.go

File:

```text
cmd/app-monitor/main.go
```

```go
package main

import (
	"log"

	"pudin.eu.org/app-monitor/internal/collector"
)

func main() {

	cpuUsage, cpuCore, err := collector.GetCPUUsage()
	if err != nil {
		log.Fatal(err)
	}

	log.Println("CPU:", cpuUsage)
	log.Println("Core:", cpuCore)
}
```

---

# Test Pertama

Jalankan:

```bash
go run ./cmd/app-monitor
```

Output:

```text
CPU: 12.5
Core: 2
```

Jika sampai tahap ini berhasil, berarti:

✅ Golang berjalan
✅ SQLite siap
✅ gopsutil berjalan
✅ Monitoring CPU berjalan

---

### Langkah Berikutnya

Setelah fondasi ini berhasil, tahap berikutnya yang paling penting adalah membuat:

1. Scheduler 10 detik
2. Simpan ke SQLite
3. Auto cleanup 24 jam
4. REST API
5. WebSocket realtime
6. Dashboard Tailwind + Chart.js

Itu akan mengubah project dari sekadar collector menjadi aplikasi monitoring yang benar-benar bisa dipakai.
