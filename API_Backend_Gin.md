Berikut **panduan praktis membuat backend API dengan Golang** dari nol, cocok untuk pemula sampai menengah 🚀
(Saya pakai **Gin Framework** karena ringan & populer)

---

## 1️⃣ Persiapan

### Install Golang

👉 [https://go.dev/dl/](https://go.dev/dl/)

Cek instalasi:

```bash
go version
```

---

## 2️⃣ Inisialisasi Project

```bash
mkdir backend-golang
cd backend-golang
go mod init backend-golang
```

Struktur folder sederhana:

```
backend-golang/
│── main.go
│── controllers/
│── models/
│── routes/
│── config/
```

---

## 3️⃣ Install Framework Gin

```bash
go get -u github.com/gin-gonic/gin
```

---

## 4️⃣ File `main.go`

```go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	r.GET("/", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "API Golang berjalan 🚀",
		})
	})

	r.Run(":8080") // localhost:8080
}
```

Jalankan:

```bash
go run main.go
```

Buka browser:

```
http://localhost:8080
```

---

## 5️⃣ Routing API (CRUD Example)

### `routes/user_routes.go`

```go
package routes

import (
	"github.com/gin-gonic/gin"
	"backend-golang/controllers"
)

func UserRoutes(router *gin.Engine) {
	user := router.Group("/users")
	{
		user.GET("/", controllers.GetUsers)
		user.POST("/", controllers.CreateUser)
	}
}
```

---

## 6️⃣ Controller

### `controllers/user_controller.go`

```go
package controllers

import (
	"net/http"
	"github.com/gin-gonic/gin"
)

type User struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

var users = []User{
	{ID: 1, Name: "Budi"},
}

func GetUsers(c *gin.Context) {
	c.JSON(http.StatusOK, users)
}

func CreateUser(c *gin.Context) {
	var newUser User
	if err := c.ShouldBindJSON(&newUser); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	users = append(users, newUser)
	c.JSON(http.StatusCreated, newUser)
}
```

---

## 7️⃣ Hubungkan Routes ke `main.go`

```go
import "backend-golang/routes"

func main() {
	r := gin.Default()
	routes.UserRoutes(r)
	r.Run(":8080")
}
```

---

## 8️⃣ Test API

### GET

```bash
curl http://localhost:8080/users
```

### POST

```bash
curl -X POST http://localhost:8080/users \
-H "Content-Type: application/json" \
-d '{"id":2,"name":"Andi"}'
```

---

## 9️⃣ Database (Opsional)

Biasanya pakai:

* **GORM** (ORM)
* PostgreSQL / MySQL / MongoDB

Install GORM:

```bash
go get -u gorm.io/gorm
go get -u gorm.io/driver/postgres
```

---

## 10️⃣ Next Level 🔥

✔ JWT Authentication
✔ Clean Architecture
✔ Middleware (Auth, Logger)
✔ Docker
✔ Unit Test

---

### ❓ Mau lanjut ke mana?

1️⃣ REST API + PostgreSQL
2️⃣ JWT Auth Login/Register
3️⃣ Clean Architecture Golang
4️⃣ Microservices Golang

Tinggal bilang mau yang mana 👍
