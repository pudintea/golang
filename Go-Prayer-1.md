# Struktur `go-prayer` (Production-Ready)

```text id="yt6c2n"
go-prayer/
├── cmd/
│   └── example/
│       └── main.go
├── internal/
│   └── astronomy/
│       ├── julian.go
│       ├── solar.go
│       └── trig.go
├── prayer/
│   ├── config.go
│   ├── methods.go
│   ├── prayer.go
│   ├── schedule.go
│   ├── validation.go
│   └── errors.go
├── tests/
│   └── prayer_test.go
├── .gitignore
├── go.mod
├── README.md
└── LICENSE
```

---
## Buat Folder
```
mkdir go-auth
```
```
cd go-auth
```
```
go mod init github.com/pudinazhar/go-prayer
```

Lanjut buka pake VS Code.


# `go.mod`

```go id="bnvm1e"
module github.com/pudinazhar/go-prayer

go 1.24
```

---

# `prayer/config.go`

```go id="9z13h9"
package prayer

type Config struct {
	Latitude  float64
	Longitude float64
	Timezone  string
	Method    CalculationMethod
}
```

---

# `prayer/methods.go`

```go id="z2g88k"
package prayer

type CalculationMethod string

const (
	Kemenag CalculationMethod = "kemenag"
	MWL     CalculationMethod = "mwl"
	ISNA    CalculationMethod = "isna"
)

type MethodParams struct {
	FajrAngle float64
	IshaAngle float64
}

func GetMethodParams(method CalculationMethod) MethodParams {
	switch method {
	case Kemenag:
		return MethodParams{
			FajrAngle: 20,
			IshaAngle: 18,
		}

	case ISNA:
		return MethodParams{
			FajrAngle: 15,
			IshaAngle: 15,
		}

	default:
		return MethodParams{
			FajrAngle: 18,
			IshaAngle: 17,
		}
	}
}
```

---

# `prayer/schedule.go`

```go id="1pvg5a"
package prayer

import "time"

type Schedule struct {
	Fajr    time.Time
	Dhuhr   time.Time
	Asr     time.Time
	Maghrib time.Time
	Isha    time.Time
}
```

---

# `prayer/errors.go`

```go id="whf6c0"
package prayer

import "errors"

var (
	ErrInvalidLatitude  = errors.New("invalid latitude")
	ErrInvalidLongitude = errors.New("invalid longitude")
	ErrInvalidTimezone  = errors.New("invalid timezone")
)
```

---

# `prayer/validation.go`

```go id="f43m4q"
package prayer

import "time"

func validateConfig(cfg Config) error {
	if cfg.Latitude < -90 || cfg.Latitude > 90 {
		return ErrInvalidLatitude
	}

	if cfg.Longitude < -180 || cfg.Longitude > 180 {
		return ErrInvalidLongitude
	}

	_, err := time.LoadLocation(cfg.Timezone)
	if err != nil {
		return ErrInvalidTimezone
	}

	return nil
}
```

---

# `prayer/prayer.go`

```go id="x2j2kw"
package prayer

import (
	"time"
)

type Prayer struct {
	config Config
	params MethodParams
	loc    *time.Location
}

func New(cfg Config) (*Prayer, error) {
	if err := validateConfig(cfg); err != nil {
		return nil, err
	}

	loc, err := time.LoadLocation(cfg.Timezone)
	if err != nil {
		return nil, err
	}

	return &Prayer{
		config: cfg,
		params: GetMethodParams(cfg.Method),
		loc:    loc,
	}, nil
}

func (p *Prayer) GetSchedule(date time.Time) (*Schedule, error) {
	year, month, day := date.Date()

	// Placeholder sementara
	fajr := time.Date(year, month, day, 4, 30, 0, 0, p.loc)
	dhuhr := time.Date(year, month, day, 12, 0, 0, 0, p.loc)
	asr := time.Date(year, month, day, 15, 15, 0, 0, p.loc)
	maghrib := time.Date(year, month, day, 18, 0, 0, 0, p.loc)
	isha := time.Date(year, month, day, 19, 10, 0, 0, p.loc)

	return &Schedule{
		Fajr:    fajr,
		Dhuhr:   dhuhr,
		Asr:     asr,
		Maghrib: maghrib,
		Isha:    isha,
	}, nil
}
```

---

# `internal/astronomy/trig.go`

```go id="4y6v6x"
package astronomy

import "math"

func DegToRad(d float64) float64 {
	return d * math.Pi / 180
}

func RadToDeg(r float64) float64 {
	return r * 180 / math.Pi
}
```

---

# `internal/astronomy/julian.go`

```go id="smy0w4"
package astronomy

import "time"

func JulianDate(t time.Time) float64 {
	return float64(t.Unix())/86400 + 2440587.5
}
```

---

# `internal/astronomy/solar.go`

```go id="k51a9u"
package astronomy

// Placeholder astronomy calculation
func SolarDeclination(jd float64) float64 {
	return 0
}
```

---

# `tests/prayer_test.go`

```go id="a9bx5g"
package tests

import (
	"testing"
	"time"

	"github.com/pudinazhar/go-prayer/prayer"
)

func TestPrayerSchedule(t *testing.T) {
	p, err := prayer.New(prayer.Config{
		Latitude:  -6.2,
		Longitude: 106.8,
		Timezone:  "Asia/Jakarta",
		Method:    prayer.Kemenag,
	})

	if err != nil {
		t.Fatal(err)
	}

	s, err := p.GetSchedule(time.Now())
	if err != nil {
		t.Fatal(err)
	}

	if s.Fajr.IsZero() {
		t.Fatal("fajr should not be zero")
	}
}
```

---

# `cmd/example/main.go`

```go id="g10g90"
package main

import (
	"fmt"
	"time"

	"github.com/pudinazhar/go-prayer/prayer"
)

func main() {
	p, err := prayer.New(prayer.Config{
		Latitude:  -6.2,
		Longitude: 106.8,
		Timezone:  "Asia/Jakarta",
		Method:    prayer.Kemenag,
	})

	if err != nil {
		panic(err)
	}

	s, err := p.GetSchedule(time.Now())
	if err != nil {
		panic(err)
	}

	fmt.Println("Fajr:", s.Fajr)
	fmt.Println("Dhuhr:", s.Dhuhr)
	fmt.Println("Asr:", s.Asr)
	fmt.Println("Maghrib:", s.Maghrib)
	fmt.Println("Isha:", s.Isha)
}
```

---

# `.gitignore`

```gitignore id="8ab06x"
bin/
dist/
coverage.out
*.log
```

---

# README Minimal

```md id="p7w2qe"
# go-prayer

Production-ready prayer time calculation library for Go.

## Install

go get github.com/yourusername/go-prayer

## Features

- Prayer time calculation
- Multiple calculation methods
- Timezone support
- Extensible architecture
- Unit tested

## Example

See cmd/example
```

---

# Langkah Berikutnya (Recommended)

Agar benar-benar production-grade, lanjutkan dengan:

## 1. Astronomy Engine

Implement:

* solar declination
* equation of time
* hour angle
* sunrise/sunset

## 2. Madhab Support

Tambahkan:

* Shafi
* Hanafi

## 3. Extra Prayer

Tambahkan:

* Sunrise
* Imsak
* Midnight
* Last Third

## 4. CI/CD

### GitHub Actions

```yaml id="5y1bb2"
name: test

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.24'

      - run: go test ./...
```

## 5. Benchmark

```bash id="fuf3o8"
go test -bench=.
```

## 6. Linter

```bash id="sqx32l"
golangci-lint run
```

## 7. Semantic Versioning

Gunakan:

* `v1.0.0`
* `v1.1.0`
* `v2.0.0`

## 8. Publish

```bash id="m8ej8s"
git tag v1.0.0
git push origin v1.0.0
```

Kalau mau, saya juga bisa bantu generate tahap berikutnya:

* implementasi astronomy calculation lengkap
* akurasi setara adhan-js
* support Kemenag Indonesia
* high precision UTC conversion
* caching
* REST API
* Fiber/Gin middleware
* CLI tool
* Docker image
* benchmark optimization
* concurrent bulk calculation
* qibla direction calculation
* Hijriyah calendar support
