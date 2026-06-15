Mantap, itu berarti:

✅ Project Golang berjalan
✅ Struktur folder benar
✅ Library `gopsutil` terpasang dengan benar
✅ CPU Core terbaca (`4 Core`)
✅ Collector berhasil dipanggil

Yang perlu diperhatikan adalah:

```text
CPU: 0
```

Ini sering terjadi karena:

```go
cpu.Percent(0, false)
```

membutuhkan interval pengukuran untuk menghitung persentase CPU.

Coba ubah file:

```go
internal/collector/cpu.go
```

menjadi:

```go
package collector

import (
	"runtime"
	"time"

	"github.com/shirou/gopsutil/v4/cpu"
)

func GetCPUUsage() (float64, int, error) {

	percent, err := cpu.Percent(time.Second, false)
	if err != nil {
		return 0, 0, err
	}

	return percent[0], runtime.NumCPU(), nil
}
```

Lalu jalankan lagi:

```bash
go run ./cmd/app-monitor
```

Biasanya hasilnya akan seperti:

```text
CPU: 2.1
Core: 4
```

atau

```text
CPU: 8.4
Core: 4
```

tergantung aktivitas server.

---

## Test Semua Collector Sekaligus

Sekarang saya sarankan upgrade `main.go` menjadi:

```go
package main

import (
	"log"

	"pudin.eu.org/app-monitor/internal/collector"
)

func main() {

	cpuUsage, cpuCore, _ := collector.GetCPUUsage()

	ramTotal, ramUsed, ramUsage, _ := collector.GetMemory()

	diskTotal, diskUsed, diskUsage, _ := collector.GetDisk()

	rx, tx, _ := collector.GetNetwork()

	log.Println("CPU Usage :", cpuUsage)
	log.Println("CPU Core  :", cpuCore)

	log.Println("RAM Total :", ramTotal)
	log.Println("RAM Used  :", ramUsed)
	log.Println("RAM Usage :", ramUsage)

	log.Println("Disk Total:", diskTotal)
	log.Println("Disk Used :", diskUsed)
	log.Println("Disk Usage:", diskUsage)

	log.Println("RX Bytes  :", rx)
	log.Println("TX Bytes  :", tx)
}
```

Agar output menjadi seperti:

```text
CPU Usage : 4.2
CPU Core  : 4

RAM Total : 4294967296
RAM Used  : 1589248000
RAM Usage : 37.1

Disk Total: 42949672960
Disk Used : 12884901888
Disk Usage: 30.0

RX Bytes  : 125463212
TX Bytes  : 78456231
```

---

## Saran Berikutnya

Sebelum masuk ke dashboard, saya ingin kita menyelesaikan **lapisan backend dulu**:

### Prioritas Berikutnya

```text
Collector
↓
SQLite
↓
Scheduler 10 detik
↓
Auto Delete > 24 jam
↓
API
↓
WebSocket
↓
Dashboard
```

Karena jika dashboard dibuat dulu, nanti akan banyak bongkar pasang.

Jadi langkah berikut yang paling tepat adalah:

### Tahap 2

Buat scheduler yang setiap 10 detik:

```text
Ambil CPU
Ambil RAM
Ambil SSD
Ambil Network
↓
Simpan ke SQLite
```

Kemudian kita cek isi database dengan:

```sql
SELECT * FROM monitor_logs;
```

Kalau data sudah masuk setiap 10 detik, berarti "mesin monitoring"-nya sudah selesai. Setelah itu dashboard hanya tinggal membaca data yang sudah tersedia.
