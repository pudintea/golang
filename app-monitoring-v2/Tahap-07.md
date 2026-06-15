Bagus, ini bukan error kode kita. Ini masalah build environment.

Error:

```text
Binary was compiled with 'CGO_ENABLED=0',
go-sqlite3 requires cgo to work.
This is a stub
```

Artinya package:

```go
github.com/mattn/go-sqlite3
```

memerlukan CGO, tetapi environment Go Anda sedang berjalan dengan:

```bash
CGO_ENABLED=0
```

---

## Cek Dulu

Jalankan:

```bash
go env CGO_ENABLED
```

Kalau keluar:

```text
0
```

berarti benar penyebabnya.

---

# Solusi 1 (Yang Saya Rekomendasikan)

Aktifkan CGO.

## Ubuntu

Install compiler:

```bash
sudo apt update
sudo apt install build-essential gcc -y
```

Cek:

```bash
gcc --version
```

Lalu:

```bash
export CGO_ENABLED=1
```

Verifikasi:

```bash
go env CGO_ENABLED
```

Harus:

```text
1
```

Kemudian:

```bash
go run ./cmd/app-monitor
```

---

# Solusi 2 (Lebih Modern dan Cocok Untuk Aplikasi Ini)

Saya justru lebih menyarankan mengganti driver SQLite.

Daripada:

```go
github.com/mattn/go-sqlite3
```

gunakan:

```go
modernc.org/sqlite
```

Keuntungan:

✅ Pure Go
✅ Tidak butuh CGO
✅ Mudah cross compile
✅ Cocok untuk binary tunggal Ubuntu
✅ Cocok untuk VPS

---

## Ganti sqlite.go

Dari:

```go
import (
	"database/sql"

	_ "github.com/mattn/go-sqlite3"
)
```

Menjadi:

```go
import (
	"database/sql"

	_ "modernc.org/sqlite"
)
```

---

Dan:

```go
func NewSQLite() (*sql.DB, error) {
	return sql.Open(
		"sqlite",
		"./data/monitor.db",
	)
}
```

---

Install:

```bash
go get modernc.org/sqlite
```

---

## Kenapa Saya Lebih Suka ModernC?

Karena nanti saat build:

```bash
CGO_ENABLED=0
go build -ldflags="-s -w"
```

tetap berhasil.

Sedangkan `go-sqlite3`:

```bash
CGO_ENABLED=0
```

akan gagal.

Untuk aplikasi monitor seperti milik Anda yang targetnya:

```text
Ubuntu
Binary tunggal
Systemd
Deploy mudah
```

saya hampir pasti memilih:

```text
modernc.org/sqlite
```

daripada:

```text
github.com/mattn/go-sqlite3
```

karena deployment jauh lebih sederhana.

---

Jadi saya sarankan sekarang:

1. Hapus dependency `github.com/mattn/go-sqlite3`
2. Ganti ke `modernc.org/sqlite`
3. Jalankan ulang
4. Setelah database berhasil dibuat, kita lanjut membuat **Network Speed Calculator (RX/TX MBps sebenarnya)** sebelum masuk ke API dan dashboard. Itu akan membuat grafik jaringan nanti jauh lebih berguna.
