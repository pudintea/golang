# Send OTP ke Email dengan Golang
Untuk kebutuhan sederhana seperti mengirim OTP, di Go saya sarankan menggunakan package **gomail** karena API-nya ringkas dan mudah digunakan.

## Install

```bash
go get gopkg.in/gomail.v2
```

## File `mailer.go`

```go
package mailer

import (
	"fmt"

	gomail "gopkg.in/gomail.v2"
)

const (
	smtpHost = "smtp.gmail.com"
	smtpPort = 587

	smtpUser = "apps@emailkamu.com"
	smtpPass = "asasasasasa"

	fromName = "APPS"
	fromMail = "apps@emailkamu.com"
)

// SendOTP mengirim email OTP
func SendOTP(email string, kodeOTP string) error {

	m := gomail.NewMessage()

	m.SetHeader("From", m.FormatAddress(fromMail, fromName))
	m.SetHeader("To", email)
	m.SetHeader("Subject", "Kode OTP Login")

	body := fmt.Sprintf(`
		<h2>Verifikasi Login</h2>

		<p>Berikut adalah kode OTP Anda:</p>

		<div style="
			font-size:32px;
			font-weight:bold;
			background:#0d6efd;
			color:white;
			padding:15px;
			width:180px;
			text-align:center;
			border-radius:8px;
			letter-spacing:5px;">
			%s
		</div>

		<p>Kode OTP berlaku beberapa menit.</p>

		<p>Jangan berikan kode ini kepada siapa pun.</p>
	`, kodeOTP)

	m.SetBody("text/html", body)

	d := gomail.NewDialer(
		smtpHost,
		smtpPort,
		smtpUser,
		smtpPass,
	)

	return d.DialAndSend(m)
}
```

---

## Cara menggunakan

Misalnya pada `main.go`

```go
package main

import (
	"fmt"

	"test-email-apps/mailer"
)

func main() {

	err := mailer.SendOTP(
		"user@gmail.com",
		"123456",
	)

	if err != nil {
		fmt.Println("Gagal mengirim email:", err)
		return
	}

	fmt.Println("Email berhasil dikirim")
}
```

---

## Versi yang lebih aman (disarankan)

Jangan menyimpan password SMTP di source code. Gunakan environment variable.

```go
package mailer

import (
	"fmt"
	"os"

	gomail "gopkg.in/gomail.v2"
)

func SendOTP(email, kodeOTP string) error {

	host := os.Getenv("MAIL_HOST")
	user := os.Getenv("MAIL_USERNAME")
	pass := os.Getenv("MAIL_PASSWORD")
	from := os.Getenv("MAIL_FROM_ADDRESS")
	fromName := os.Getenv("MAIL_FROM_NAME")

	m := gomail.NewMessage()

	m.SetHeader("From", m.FormatAddress(from, fromName))
	m.SetHeader("To", email)
	m.SetHeader("Subject", "Kode OTP Login")

	body := fmt.Sprintf(`
		<h2>Verifikasi Login</h2>
		<p>Kode OTP Anda:</p>
		<h1>%s</h1>
	`, kodeOTP)

	m.SetBody("text/html", body)

	d := gomail.NewDialer(host, 587, user, pass)

	return d.DialAndSend(m)
}
```

Kemudian jalankan dengan environment variable seperti:

```bash
export MAIL_HOST=smtp.gmail.com
export MAIL_USERNAME=apps@emailkamu.com
export MAIL_PASSWORD=asasasasasa
export MAIL_FROM_ADDRESS=apps@emailkamu.com
export MAIL_FROM_NAME="APPS"
```

Atau gunakan file `.env` dengan library seperti `github.com/joho/godotenv`.

Versi ini lebih cocok jika nantinya aplikasi Anda akan di-deploy ke VPS, Docker, atau Kubernetes karena kredensial tidak perlu ditulis langsung di kode.
