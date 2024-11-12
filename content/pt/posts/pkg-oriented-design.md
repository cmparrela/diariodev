---
date: '2024-11-11T21:09:33-03:00'
title: 'Organizando um projeto em Go orientado a pacotes'
showToc: false
---

Neste post, vamos explorar como organizar um projeto em Go. Diferente de outras linguagens, onde frameworks geralmente oferecem padrões de organização de código, em Go temos liberdade para organizar o projeto da maneira que preferirmos. Isso é bom, mas sem um padrão claro, o código pode se tornar desorganizado e difícil de manter. Por isso, é essencial estabelecer uma estrutura que permita aos desenvolvedores localizar facilmente onde cada parte do código deve ser colocada.

Nessa estrutura vamos conseguir aplicar os principios de Clean Architecture e usar as vantagens da linguagem Go para criar um projeto bem organizado e fácil de manter. Vale ressaltar que a estrutura deve mudar de acordo com as necessidades de cada projeto, então é importante adaptar para o contexto específico.

Go é uma linguagem organizada em pacotes, e por isso é fundamental pensar na estrutura do projeto de forma que os pacotes sejam bem definidos e de fácil entendimento. Algumas diretrizes ajudam a manter a organização e a qualidade do código:

- Isolamento de Responsabilidade: Cada pacote deve ter uma responsabilidade bem definida, evitando misturar funcionalidades. Isso torna o código mais modular e fácil de compreender.
- Minimização de Dependências: Os pacotes devem depender de outros pacotes apenas quando realmente necessário. Em Go, podemos cair em `circular dependency` se não tomarmos cuidado com as dependências.
- Estrutura Padronizada: Utilizar uma estrutura previsível facilita o entendimento e a navegação pelo código, especialmente em equipes.
- Testabilidade: Organizar o código de forma a facilitar a criação de testes unitários e de integração.
- Inversão de Dependência: Usar interfaces para abstrair dependências, especialmente pacotes externos, mantendo o código mais flexível e desacoplado.
- Independência de Pacotes de Terceiros: Evite dependências diretas de pacotes externos, utilizando interfaces e sempre que possível, criando wrappers para interagir com essas dependências.

# cmd

Vamos começar pela pasta `cmd`, onde fica o código principal da aplicação, o arquivo `main.go`. Esse arquivo é responsável por integrar todas as dependências do projeto e gerar o executável. Podemos criar subpastas dentro de cmd para cada aplicação executável do projeto, como `api` e `worker`, organizando os pontos de entrada de cada serviço.

```bash
├── cmd
│   ├── api
│   │   └── main.go
│   └── worker
│       └── main.go 
```

# domain

**Tipos de dominio (domain types)**

A pasta `domain` é onde ficam os tipos de domínio da aplicação. Tipos de domínio são estruturas de dados que representam os conceitos principais do negócio da aplicação. Por exemplo, em um sistema de vendas, os tipos de domínio poderiam incluir `Order`, `Product`, `Customer`. Esses tipos refletem as entidades principais da aplicação.

```bash
├── domain
│   ├── product.go
│   └── order.go
```

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

# internal
A pasta `internal` é onde fica o código que não deve ser acessado externamente. O próprio Go impede que pacotes dentro da pasta internal possam ser acessados por outros arquivos fora do mesmo pacote `main`. Eu gosto de usar essa pasta para colocar a lógica de negócio da minha aplicação, separando-a em subpastas por domínio (domain types).

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

Observe que aqui é onde a lógica de negócio da aplicação é implementada. Cada tipo de domínio tem um arquivo `service.go` que contém a lógica de negócio para esse tipo. O arquivo `repository.go` contém a lógica de acesso a dados para esse tipo. O arquivo `handler.go` contém a lógica de manipulação de solicitações HTTP para esse tipo. O arquivo `dto.go` contém os tipos de dados de transferência que são usados para representar os dados que são passados entre as camadas.

Se precisarmos fazer uma validaçao de dados de input para POST e PUT devemos fazer no arquivo de DTO.

Se precisarmos criar uma struct para representar os querys params, devemos fazer no arquivo de DTO.

# config

A pasta `config` é onde ficam os arquivos de configuração da aplicação.

```bash
├── config
│   └── config.go
```

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

# pkg
A pasta `pkg` normalmente é utilizada para colocar lib interna, wrappers e código que pode ser reutilizado em outros projetos, por exemplo, um pacote de log, um pacote de conexão com banco de dados, etc

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

# Estrutura completa

Aqui está a estrutura completa do projeto:

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