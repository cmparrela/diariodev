---
date: '2024-11-12T09:02:00-03:00'
title: 'Organizing Go Projects with Package Oriented Design'
showToc: false
---

In this post, we will explore how to organize a Go project. Unlike other languages where frameworks often provide code organization patterns, in Go we have the freedom to organize the project as we prefer. This is good, but without a clear pattern, the code can become disorganized and difficult to maintain. Therefore, it is essential to establish a structure that allows developers to easily locate where each part of the code should be placed.

In this structure, we will be able to apply the principles of Clean Architecture and use the advantages of the Go language to create a well-organized and easy-to-maintain project. It is worth noting that the structure should change according to the needs of each project, so it is important to adapt to the specific context.

Go is a language organized in packages, and therefore it is essential to think about the project structure so that the packages are well-defined and easy to understand. Some guidelines help maintain code organization and quality:

- Responsibility Isolation: Each package should have a well-defined responsibility, avoiding mixing functionalities. This makes the code more modular and easier to understand.
- Dependency Minimization: Packages should depend on other packages only when really necessary. In Go, we can fall into `circular dependency` if we are not careful with dependencies.
- Standardized Structure: Using a predictable structure makes it easier to understand and navigate the code, especially in teams.
- Testability: Organize the code to facilitate the creation of unit and integration tests.
- Dependency Inversion: Use interfaces to abstract dependencies, especially external packages, keeping the code more flexible and decoupled.
- Independence from Third-Party Packages: Avoid direct dependencies on external packages, using interfaces and whenever possible, creating wrappers to interact with these dependencies.

## cmd

Let's start with the `cmd` folder, where the main application code, the `main.go` file, is located. This file is responsible for integrating all project dependencies and generating the executable. We can create subfolders within cmd for each executable application in the project, such as `api` and `worker`, organizing the entry points of each service.

```bash
├── cmd
│   ├── api
│   │   └── main.go
│   └── worker
│       └── main.go 
```

## domain

### Domain (domain types)

The `domain` folder is where the application's domain types are located. Domain types are data structures that represent the main business concepts of the application. For example, in a sales system, domain types could include `Order`, `Product`, `Customer`. These types reflect the main entities of the application.

So, let's create the domain types for our project as follows:
```bash
├── domain
│   ├── product.go
│   └── order.go
```

The code inside each file would be something like this:
```go
// domain/product.go
package domain

import (
	"time"

	"go.mongodb.org/mongo-driver/bson/primitive"
)

type Product struct {
	ID          primitive.ObjectID `json:"id" bson:"_id,omitempty"`
	Name        string             `json:"name" bson:"name"`
	Description string             `json:"description" bson:"description"`
	Price       float64            `json:"price" bson:"price"`
	UpdatedAt   time.Time          `json:"updated_at" bson:"updated_at"`
	CreatedAt   time.Time          `json:"created_at" bson:"created_at"`
}
```

```go
// domain/order.go
package domain

import (
	"time"

	"go.mongodb.org/mongo-driver/bson/primitive"
)

type OrderStatus string

const (
	OrderStatusPending  OrderStatus = "pending"
	OrderStatusApproved OrderStatus = "approved"
)

type Order struct {
	ID        primitive.ObjectID `json:"id" bson:"_id,omitempty"`
	Customer  string             `json:"customer" bson:"customer"`
	Products  []Product          `json:"products" bson:"products"`
	Total     float64            `json:"total" bson:"total"`
	Status    OrderStatus        `json:"status" bson:"status"`
	UpdatedAt time.Time          `json:"updated_at" bson:"updated_at"`
	CreatedAt time.Time          `json:"created_at" bson:"created_at"`
}

```

## internal
The `internal` folder is where the code that should not be accessed externally is located. Go itself prevents packages within the internal folder from being accessed by other files outside the same `main` package. I like to use this folder to place the business logic of my application, separating it into subfolders by domain (domain types).

```bash
├── internal
│    ├── order
│    │   ├── dto.go
│    │   ├── service.go 
│    │   ├── repository.go
│    │   └── handler.go
│    └── product
│        ├── dto.go
│        ├── service.go 
│        ├── repository.go
│        └── handler.go
```

Note that this is where the business logic of the application is implemented. Each domain type has a `service.go` file that contains the business logic for that type. The `repository.go` file contains the data access logic for that type. The `handler.go` file contains the HTTP request handling logic for that type. The `dto.go` file contains the data transfer types that are used to represent the data that is passed between layers.

### DTO

Although we have the domain folder with the data structures that we will store in the database, we may need a different data structure to represent the data we want to treat as editable in our API. For this, we can use the `dto.go` file to create these structures. In the case of a `PUT` or `POST` endpoint, I add the `Input` suffix, so for an order creation function, I would have something like this:

```go
// internal/order/dto.go
type CreateInput struct {
	Customer string  `json:"customer" validate:"required"`
	Total    float64 `json:"total" validate:"required"`
	Status   string  `json:"status" validate:"required,oneof=pending approved"`
}
```

In the case of a `GET` endpoint for those that will have query params, I add the `Params` suffix, so for an order listing function, I would have something like this:

```go
// internal/order/dto.go
type ListParams struct {
	Customer string `json:"customer"`
}
```

To define the response payload of the endpoints, you can create a new struct specific to the return. In this case, I use the `Response` suffix.

```go
// internal/order/dto.go
type CreateResponse struct {
	Customer string      `json:"customer"`
	Products []Product   `json:"products"`
	Total    float64     `json:"total"`
	Status   OrderStatus `json:"status"`
}
```

### Service

The service is where the business logic is implemented. It is responsible for orchestrating calls to the repository, validating input and output data, and executing the business logic.

```go
// internal/order/service.go
package order

import (
	"context"

	"github.com/cmparrela/ddev-pkg-oriented-design/domain"
	"github.com/go-playground/validator/v10"
)

type Service interface {
	Create(ctx context.Context, input CreateInput) (CreateResponse, error)
}

type service struct {
	validator  validator.Validate
	repository Repository
}

func NewService(validator validator.Validate, repository Repository) Service {
	return &service{
		validator:  validator,
		repository: repository,
	}
}

func (s *service) Create(ctx context.Context, input CreateInput) (*domain.Order, error) {
	err := s.validator.Struct(input)
	if err != nil {
		return nil, err
	}

order := domain.Order{
		Customer: input.Customer,
		Total:    input.Total,
		Status:   domain.OrderStatus(input.Status),
	}

	order, err := s.repository.Create(ctx, order)
	if err != nil {
		return nil, err
	}

	return order, nil
}
```

## config

The `config` folder is where the application's configuration files are located.

```bash
├── config
│   └── config.go
```

Example configuration file:
```go
//config/config.go
package config

import (
	"github.com/kelseyhightower/envconfig"
)

type Config struct {
	MongoURI          string `envconfig:"mongo_uri" default:"mongodb://localhost:27017"`
	MongoDatabaseName string `envconfig:"mongo_database_name" default:"app-example"`
}

func New() (cfg Config, err error) {
	err = envconfig.Process("", &cfg)
	return
}

```

## pkg
The `pkg` folder is usually used to place internal libraries, wrappers, and code that can be reused in other projects, for example, a logging package, a database connection package, etc.

```bash
├── pkg
│   ├── httpclient
│   │   └── client.go
│   ├── mongodb
│   │   └── mongo.go
│   ├── validator
│   │   └── validator.go
│   └── logger
│       └── logger.go
```

## Complete Structure

Here is the complete project structure:

```bash
├── cmd
│   ├── api
│   │   └── main.go
│   ├── worker
│   │   └── main.go 
│   
├── domain
│   ├── product.go
│   ├── order.go
│   
├── internal
│   ├── order
│   │   ├── dto.go
│   │   ├── service.go 
│   │   ├── repository.go
│   │   └── handler.go
│   ├── product
│   │   ├── dto.go
│   │   ├── service.go 
│   │   ├── repository.go
│   │   └── handler.go
│   
├── config
│   ├── config.go
│   
├── pkg
│   ├── httpclient
│   │   └── client.go
│   ├── mongodb
│   │   └── mongo.go
│   ├── validator
│   │   └── validator.go
│   ├── logger
│   │   └── logger.go
```
