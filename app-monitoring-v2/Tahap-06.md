Sip. Sekarang kita masuk ke fase yang menurut saya paling penting: **membuat engine monitoring yang benar-benar menyimpan data ke SQLite setiap 10 detik**, lengkap dengan tabel `server_info`.

Kalau tahap ini selesai, dashboard nanti tinggal membaca data yang sudah ada.

---

# 1. Upgrade Database Schema

Buka:

```go
internal/database/migration.go
```

Ganti menjadi:

```go
package database

import "database/sql"

func Migrate(db *sql.DB) error {

	query := `
	CREATE TABLE IF NOT EXISTS server_info (
		id INTEGER PRIMARY KEY AUTOINCREMENT,

		hostname TEXT,
		os TEXT,
		kernel TEXT,
		arch TEXT,

		created_at DATETIME DEFAULT CURRENT_TIMESTAMP
	);

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

# 2. Buat Collector Informasi Server

Buat folder:

```text
internal/system/
```

---

## hostname.go

```go
package system

import "os"

func Hostname() string {

	name, err := os.Hostname()
	if err != nil {
		return "unknown"
	}

	return name
}
```

---

## kernel.go

```go
package system

import (
	"runtime"
)

func Kernel() string {
	return runtime.GOOS
}
```

---

## arch.go

```go
package system

import "runtime"

func Arch() string {
	return runtime.GOARCH
}
```

---

## os.go

```go
package system

import (
	"os"
	"strings"
)

func OSVersion() string {

	data, err := os.ReadFile("/etc/os-release")
	if err != nil {
		return "Linux"
	}

	lines := strings.Split(string(data), "\n")

	for _, line := range lines {

		if strings.HasPrefix(line, "PRETTY_NAME=") {

			return strings.Trim(
				strings.TrimPrefix(line, "PRETTY_NAME="),
				`"`,
			)
		}
	}

	return "Linux"
}
```

---

# 3. Repository Server Info

Buat:

```text
internal/database/server_info.go
```

```go
package database

import "database/sql"

func SaveServerInfo(
	db *sql.DB,
	hostname string,
	osName string,
	kernel string,
	arch string,
) error {

	var count int

	err := db.QueryRow(
		"SELECT COUNT(*) FROM server_info",
	).Scan(&count)

	if err != nil {
		return err
	}

	if count > 0 {
		return nil
	}

	_, err = db.Exec(`
	INSERT INTO server_info(
		hostname,
		os,
		kernel,
		arch
	)
	VALUES(?,?,?,?)
	`,
		hostname,
		osName,
		kernel,
		arch,
	)

	return err
}
```

---

# 4. Repository Monitor Logs

Buat:

```text
internal/database/monitor_logs.go
```

```go
package database

import (
	"database/sql"
	"time"

	"pudin.eu.org/app-monitor/internal/models"
)

func InsertMonitorLog(
	db *sql.DB,
	logData models.MonitorLog,
) error {

	_, err := db.Exec(`
	INSERT INTO monitor_logs(
		created_at,

		cpu_usage,
		cpu_core,

		ram_total,
		ram_used,
		ram_usage,

		disk_total,
		disk_used,
		disk_usage,

		rx_bytes,
		tx_bytes,

		rx_speed,
		tx_speed
	)
	VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?)
	`,
		time.Now(),

		logData.CPUUsage,
		logData.CPUCore,

		logData.RAMTotal,
		logData.RAMUsed,
		logData.RAMUsage,

		logData.DiskTotal,
		logData.DiskUsed,
		logData.DiskUsage,

		logData.RXBytes,
		logData.TXBytes,

		logData.RXSpeed,
		logData.TXSpeed,
	)

	return err
}
```

---

# 5. Auto Cleanup Data > 24 Jam

Buat:

```text
internal/database/cleanup.go
```

```go
package database

import "database/sql"

func CleanupOldData(db *sql.DB) error {

	_, err := db.Exec(`
	DELETE FROM monitor_logs
	WHERE created_at < datetime('now','-24 hours')
	`)

	return err
}
```

---

# 6. Scheduler Monitoring

Buat:

```text
internal/scheduler/collector.go
```

Versi pertama:

```go
package scheduler

import (
	"log"
	"time"

	"database/sql"

	"pudin.eu.org/app-monitor/internal/collector"
	"pudin.eu.org/app-monitor/internal/database"
	"pudin.eu.org/app-monitor/internal/models"
)

func StartCollector(db *sql.DB) {

	ticker := time.NewTicker(
		10 * time.Second,
	)

	defer ticker.Stop()

	for {

		cpuUsage, cpuCore, _ :=
			collector.GetCPUUsage()

		ramTotal, ramUsed, ramUsage, _ :=
			collector.GetMemory()

		diskTotal, diskUsed, diskUsage, _ :=
			collector.GetDisk()

		rx, tx, _ :=
			collector.GetNetwork()

		data := models.MonitorLog{
			CPUUsage: cpuUsage,
			CPUCore:  cpuCore,

			RAMTotal: ramTotal,
			RAMUsed:  ramUsed,
			RAMUsage: ramUsage,

			DiskTotal: diskTotal,
			DiskUsed:  diskUsed,
			DiskUsage: diskUsage,

			RXBytes: rx,
			TXBytes: tx,

			RXSpeed: 0,
			TXSpeed: 0,
		}

		err := database.InsertMonitorLog(
			db,
			data,
		)

		if err != nil {
			log.Println(err)
		} else {
			log.Println("saved")
		}

		<-ticker.C
	}
}
```

---

# 7. Update Main.go

```go
package main

import (
	"log"

	"pudin.eu.org/app-monitor/internal/database"
	"pudin.eu.org/app-monitor/internal/scheduler"
	"pudin.eu.org/app-monitor/internal/system"
)

func main() {

	db, err := database.NewSQLite()
	if err != nil {
		log.Fatal(err)
	}

	defer db.Close()

	err = database.Migrate(db)
	if err != nil {
		log.Fatal(err)
	}

	err = database.SaveServerInfo(
		db,
		system.Hostname(),
		system.OSVersion(),
		system.Kernel(),
		system.Arch(),
	)

	if err != nil {
		log.Fatal(err)
	}

	log.Println("Aplikasi Monitor Started")

	scheduler.StartCollector(db)
}
```

---

# Test

Jalankan:

```bash
go run ./cmd/app-monitor
```

Harus muncul:

```text
2026/06/15 11:30:00 Aplikasi Monitor Started
2026/06/15 11:30:10 saved
2026/06/15 11:30:20 saved
2026/06/15 11:30:30 saved
```

Kemudian cek database:

```bash
sqlite3 data/monitor.db
```

Lihat tabel:

```sql
.tables
```

Harus ada:

```text
monitor_logs
server_info
```

Cek jumlah data:

```sql
SELECT COUNT(*) FROM monitor_logs;
```

Jika angkanya bertambah setiap 10 detik, maka fondasi monitoring dan penyimpanan data sudah berhasil. Setelah itu langkah berikutnya adalah **menghitung RX/TX speed yang sebenarnya (bukan total bytes)** dan membuat endpoint API `/api/summary` serta `/api/history`.
