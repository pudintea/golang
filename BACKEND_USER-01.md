# Membuat backend dengan golang dengan Feature-Based / Modular Architecture
Bagus. Kalau tujuanmu belajar untuk project yang nantinya bisa berkembang menjadi **Auth, User, Product, Order, Payment, dll**, saya lebih menyarankan struktur seperti ini daripada yang sebelumnya.

Struktur ini masih cukup mudah dipahami pemula, tetapi sudah mendekati pola yang sering dipakai di perusahaan.

---

## Konsep Architecture

Setiap fitur berdiri sendiri di dalam folder `modules`.

Misalnya:

```text
modules/
├── auth/
├── users/
├── products/
├── orders/
```

Masing-masing fitur memiliki:

```text
controller
service
repository
dto
entity
routes
```

Alur request:

```text
Request
   ↓
Controller
   ↓
Service
   ↓
Repository
   ↓
Database
```

---

## Struktur Folder Lengkap

```text
go-api-user/

├── cmd/
│   └── api/
│       └── main.go
│
├── config/
│   ├── database.go
│   └── env.go
│
├── modules/
│   │
│   └── users/
│       │
│       ├── controller/
│       │   └── user_controller.go
│       │
│       ├── service/
│       │   └── user_service.go
│       │
│       ├── repository/
│       │   └── user_repository.go
│       │
│       ├── dto/
│       │   ├── create_user_request.go
│       │   └── user_response.go
│       │
│       ├── entity/
│       │   └── user.go
│       │
│       ├── routes/
│       │   └── user_routes.go
│       │
│       └── module.go
│
├── middlewares/
│   ├── error_middleware.go
│   └── notfound_middleware.go
│
├── utils/
│   ├── response.go
│   └── bcrypt.go
│
├── .env
├── go.mod
└── go.sum
```

---

## Folder Config

Berisi konfigurasi global.

```text
config/
├── database.go
└── env.go
```

---

## config/env.go

```go
package config

import (
	"log"

	"github.com/joho/godotenv"
)

func LoadEnv() {
	err := godotenv.Load()

	if err != nil {
		log.Fatal(".env not found")
	}
}
```

---

## config/database.go

```go
package config

import (
	"fmt"
	"os"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

var DB *gorm.DB

func ConnectDatabase() {

	dsn := fmt.Sprintf(
		"%s:%s@tcp(%s:%s)/%s?charset=utf8mb4&parseTime=True&loc=Local",
		os.Getenv("DB_USER"),
		os.Getenv("DB_PASSWORD"),
		os.Getenv("DB_HOST"),
		os.Getenv("DB_PORT"),
		os.Getenv("DB_NAME"),
	)

	db, err := gorm.Open(
		mysql.Open(dsn),
		&gorm.Config{},
	)

	if err != nil {
		panic(err)
	}

	DB = db
}
```

---

## Utils

### utils/response.go

```go
package utils

import "github.com/gin-gonic/gin"

func Success(
	c *gin.Context,
	data interface{},
) {

	c.JSON(200, gin.H{
		"success": true,
		"data": data,
	})
}

func Created(
	c *gin.Context,
	message string,
) {

	c.JSON(201, gin.H{
		"success": true,
		"message": message,
	})
}

func Error(
	c *gin.Context,
	code int,
	message string,
) {

	c.JSON(code, gin.H{
		"success": false,
		"message": message,
	})
}
```

---

### utils/bcrypt.go

```go
package utils

import "golang.org/x/crypto/bcrypt"

func HashPassword(
	password string,
) (string, error) {

	hash, err := bcrypt.GenerateFromPassword(
		[]byte(password),
		bcrypt.DefaultCost,
	)

	return string(hash), err
}
```

---

## User Module

Sekarang kita masuk ke fitur User.

---

## Entity Layer

Entity adalah representasi tabel database.

```text
entity/
└── user.go
```

### entity/user.go

```go
package entity

type User struct {
	ID       uint   `gorm:"primaryKey"`
	Name     string `gorm:"not null"`
	Email    string `gorm:"unique"`
	Password string `gorm:"not null"`
}
```

---

## DTO Layer

DTO = Data Transfer Object

Digunakan untuk request dan response.

---

### dto/create_user_request.go

```go
package dto

type CreateUserRequest struct {
	Name     string `json:"name"`
	Email    string `json:"email"`
	Password string `json:"password"`
}
```

---

### dto/user_response.go

```go
package dto

type UserResponse struct {
	ID    uint   `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}
```

---

## Repository Layer

Repository hanya berurusan dengan database.

```text
repository/
└── user_repository.go
```

### repository/user_repository.go

```go
package repository

import (
	"go-api-user/config"
	"go-api-user/modules/users/entity"
)

type UserRepository struct{}

func NewUserRepository() *UserRepository {
	return &UserRepository{}
}

func (r *UserRepository) FindAll() (
	[]entity.User,
	error,
) {

	var users []entity.User

	err := config.DB.
		Find(&users).
		Error

	return users, err
}

func (r *UserRepository) Create(
	user *entity.User,
) error {

	return config.DB.
		Create(user).
		Error
}
```

---

## Service Layer

Tempat business logic.

```text
service/
└── user_service.go
```

### service/user_service.go

```go
package service

import (
	"go-api-user/modules/users/dto"
	"go-api-user/modules/users/entity"
	"go-api-user/modules/users/repository"
	"go-api-user/utils"
)

type UserService struct {
	repository *repository.UserRepository
}

func NewUserService(
	repository *repository.UserRepository,
) *UserService {

	return &UserService{
		repository,
	}
}

func (s *UserService) GetUsers() (
	[]dto.UserResponse,
	error,
) {

	users, err :=
		s.repository.FindAll()

	if err != nil {
		return nil, err
	}

	var result []dto.UserResponse

	for _, user := range users {

		result = append(result,
			dto.UserResponse{
				ID: user.ID,
				Name: user.Name,
				Email: user.Email,
			},
		)
	}

	return result, nil
}

func (s *UserService) CreateUser(
	req dto.CreateUserRequest,
) error {

	hash, err := utils.HashPassword(
		req.Password,
	)

	if err != nil {
		return err
	}

	user := entity.User{
		Name: req.Name,
		Email: req.Email,
		Password: hash,
	}

	return s.repository.Create(&user)
}
```

---

## Controller Layer

Controller menerima HTTP Request.

```text
controller/
└── user_controller.go
```

### controller/user_controller.go

```go
package controller

import (
	"go-api-user/modules/users/dto"
	"go-api-user/modules/users/service"
	"go-api-user/utils"
	"net/http"

	"github.com/gin-gonic/gin"
)

type UserController struct {
	service *service.UserService
}

func NewUserController(
	service *service.UserService,
) *UserController {

	return &UserController{
		service,
	}
}

func (ctrl *UserController) FindAll(
	c *gin.Context,
) {

	users, err :=
		ctrl.service.GetUsers()

	if err != nil {

		utils.Error(
			c,
			http.StatusInternalServerError,
			err.Error(),
		)

		return
	}

	utils.Success(c, users)
}

func (ctrl *UserController) Create(
	c *gin.Context,
) {

	var req dto.CreateUserRequest

	err := c.ShouldBindJSON(&req)

	if err != nil {

		utils.Error(
			c,
			http.StatusBadRequest,
			err.Error(),
		)

		return
	}

	err = ctrl.service.CreateUser(req)

	if err != nil {

		utils.Error(
			c,
			http.StatusInternalServerError,
			err.Error(),
		)

		return
	}

	utils.Created(
		c,
		"User created successfully",
	)
}
```

---

## Routes Layer

```text
routes/
└── user_routes.go
```

### routes/user_routes.go

```go
package routes

import (
	"go-api-user/modules/users/controller"

	"github.com/gin-gonic/gin"
)

func RegisterUserRoutes(
	router *gin.Engine,
	controller *controller.UserController,
) {

	users := router.Group("/users")

	{
		users.GET("", controller.FindAll)
		users.POST("", controller.Create)
	}
}
```

---

## Module Registration

Ini bagian yang sering dipakai pada project besar.

Tujuannya supaya `main.go` tidak berisi banyak kode.

```text
module.go
```

### modules/users/module.go

```go
package users

import (
	"go-api-user/modules/users/controller"
	"go-api-user/modules/users/repository"
	"go-api-user/modules/users/routes"
	"go-api-user/modules/users/service"

	"github.com/gin-gonic/gin"
)

func RegisterModule(
	router *gin.Engine,
) {

	userRepository :=
		repository.NewUserRepository()

	userService :=
		service.NewUserService(
			userRepository,
		)

	userController :=
		controller.NewUserController(
			userService,
		)

	routes.RegisterUserRoutes(
		router,
		userController,
	)
}
```

---

## Main.go

Sekarang main.go sangat bersih.

### cmd/api/main.go

```go
package main

import (
	"go-api-user/config"
	"go-api-user/modules/users"
	"go-api-user/modules/users/entity"

	"github.com/gin-gonic/gin"
)

func main() {

	config.LoadEnv()

	config.ConnectDatabase()

	config.DB.AutoMigrate(
		&entity.User{},
	)

	router := gin.Default()

	users.RegisterModule(router)

	router.Run(":3000")
}
```

---

## Hasil Akhir Dependency Flow

Saat aplikasi start:

```text
main.go
    ↓
RegisterModule()
    ↓
Repository
    ↓
Service
    ↓
Controller
    ↓
Routes
```

Saat request masuk:

```text
POST /users
    ↓
Controller
    ↓
Service
    ↓
Repository
    ↓
MySQL
```

---

Untuk tahap berikutnya, saya biasanya menambahkan beberapa hal agar benar-benar production-ready:

```text
shared/
├── validator/
├── logger/
├── pagination/
├── database/
├── response/
├── constants/
└── helpers/

modules/
├── auth/
├── users/
├── products/
├── orders/
└── payments/
```

Lalu menambahkan:

* JWT Authentication
* Custom Error Handling
* Request Validation (`validator/v10`)
* Pagination
* Dependency Injection
* Environment Config Struct
* Transaction Management
* Unit Testing

Itu sudah sangat dekat dengan struktur backend Go yang umum dipakai di startup maupun perusahaan skala menengah.
