---
date: '2024-10-05T21:09:33-03:00'
title: 'Criando API Rest com Golang e AWS Lambda'
---

Neste post, vamos explorar como construir uma API REST com Golang e fazer o deploy no AWS Lambda, aproveitando os benefícios do serverless para criar uma aplicação escalável e econômica.

Uma das grandes vantagens de usar o AWS Lambda é a redução de custos, especialmente com o plano gratuito da AWS, além da diminuição no tempo de desenvolvimento e manutenção.

## O que é AWS Lambda

O AWS Lambda é o serviço serverless da AWS. No modelo *serverless*, você não precisa gerenciar diretamente os servidores onde sua aplicação é executada. No caso do AWS Lambda, a AWS cuida de toda a infraestrutura, escalando automaticamente sua aplicação conforme necessário. Isso significa que você paga apenas pelo tempo de execução da sua função, o que é ideal para aplicações com cargas variáveis ou intermitentes.

Com o serverless, você pode focar exclusivamente na lógica de negócio, sem se preocupar com manutenção de servidores ou configurações complexas de escalabilidade. O AWS Lambda se encarrega da administração dos recursos, liberando você para trabalhar nas funcionalidades da aplicação.

Além disso, o custo é um grande atrativo. Em muitos casos, o AWS Lambda pode ser gratuito, graças ao generoso plano gratuito da AWS. Esse plano cobre uma quantidade considerável de execuções por mês, o que torna o Lambda uma ótima opção para testes, desenvolvimento e até mesmo para aplicações de produção com uso moderado.

### Free Tier

A AWS oferece um plano gratuito para o AWS Lambda que inclui 1 milhão de requisições e 400.000 GB-segundos de computação por mês. Isso significa que você pode processar até 1 milhão de requisições mensais sem custos adicionais, o que torna o Lambda muito atraente para pequenos projetos e para fins de teste.

### Cuidados com o Custo

Apesar do plano gratuito, é importante monitorar o uso do AWS Lambda para evitar custos inesperados. O ideal é sempre acompanhar o consumo e configurar alertas para ser notificado caso o uso ultrapasse o limite gratuito.

### Como Criar um Budget no AWS Budgets

Para garantir que você mantenha controle sobre os custos, é recomendável configurar um budget na AWS. O AWS Budgets permite definir um limite de gastos e receber alertas quando o uso se aproxima desse limite, ajudando a evitar surpresas na fatura.

1. **Acesse o AWS Budgets**  
   No Console AWS, vá para a seção de "Billing" e clique em "Budgets" no menu à esquerda.

2. **Crie um Novo Budget**  
   Clique em "Create a budget" e selecione o tipo de orçamento ("Cost Budget") para definir um limite baseado em valor.

3. **Defina o Valor e as Configurações de Alerta**  
   Insira o valor limite que você deseja para o mês e configure notificações por email para receber alertas quando o consumo atingir, por exemplo, 80% e 100% do budget.

Com essas configurações, você pode utilizar o AWS Lambda com mais segurança, aproveitando o plano gratuito sem correr o risco de custos inesperados.


## Preparando o ambiente

Para fazer o deploy da nossa API no AWS Lambda, precisamos configurar algumas ferramentas essenciais. Vamos instalar o AWS CLI e o AWS SAM, além de criar um usuário com permissões adequadas na AWS.

### 1. Instalando o AWS CLI
O AWS CLI é uma ferramenta de linha de comando que permite interagir com os serviços da AWS diretamente do terminal.

Para instalar o AWS CLI, siga as instruções na documentação oficial:

[Documentação para Instalação do AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

Após a instalação, verifique se o AWS CLI foi instalado corretamente com o comando:

```bash
aws --version
```

### 2. Instalando o AWS SAM

O AWS SAM pode ser considerado uma forma de Infraestrutura como Código (IaC). Com ele, você descreve a infraestrutura da sua aplicação serverless utilizando um arquivo de configuração YAML. Isso permite versionar e automatizar o processo de deploy da infraestrutura de forma consistente e repetível, sem a necessidade de configurações manuais.

Para instalar o AWS SAM, siga as instruções na documentação oficial:

[Documentação para  Instalação do AWS SAM](https://docs.aws.amazon.com/pt_br/serverless-application-model/latest/developerguide/install-sam-cli.html)


### 3. Configurando um Usuário na AWS

Para realizar o deploy, vamos precisar de um usuário na AWS com as permissões adequadas.
1. Acesse o console do IAM na AWS.
2. Clique em `Adicionar usuário`.
3. Escolha um nome para o usuário e marque a opção de `Acesso programático`.
4. Em `Permissões`, selecione `Anexar políticas diretamente` e escolha a política de `Administrador`.
5. Conclua a criação e guarde as credenciais de acesso.

Agora, configure o AWS CLI com o comando abaixo e insira as credenciais do usuário:
```bash
aws configure
```

## Criando a API Rest com Golang

### 1. Iniciando o projeto
Para esse projeto vamos usar o `aws-lambda-go-api-proxy` é uma biblioteca que facilita a criação de APIs REST em Golang, permitindo que você utilize frameworks como o `Gin` ou `Fiber` para desenvolver suas APIs de forma mais estruturada e eficiente. A grande vantagem é que, com poucas linhas de código, você consegue alternar entre o uso do framework local e o AWS Lambda, facilitando a migração caso decida parar de usar o Lambda no futuro.

Você pode acessar a documentação oficial do projeto neste link:  
[Documentação aws-lambda-go-api-proxy](https://github.com/awslabs/aws-lambda-go-api-proxy)

Neste post, vamos mostrar um exemplo utilizando o framework Fiber.

### 2. Criando a API com Golang e Fiber

No arquivo `main.go`, adicione o seguinte código para criar uma rota simples que retorna um JSON com a mensagem "Hello, World!":
```golang
// main.go
package main

import (
	"context"
	"log"
	"os"

	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
	fiberadapter "github.com/awslabs/aws-lambda-go-api-proxy/fiber"
	"github.com/gofiber/fiber/v2"
	"github.com/joho/godotenv"
)

var fiberLambda *fiberadapter.FiberLambda
var app *fiber.App

func init() {
	err := godotenv.Load()
	if err != nil {
		log.Fatalf("error loading .env file: %v", err)
	}

	app = fiber.New()

	app.Get("/", func(c *fiber.Ctx) error {
		return c.SendString("Hello, World!")
	})

	fiberLambda = fiberadapter.New(app)
}

func Handler(ctx context.Context, req events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
	return fiberLambda.ProxyWithContext(ctx, req)
}

func main() {
	if os.Getenv("LAMBDA_ENABLED") == "true" {
		lambda.Start(Handler)
	} else {
		err := app.Listen(":3000")
		if err != nil {
			panic(err)
		}
	}
}
```

Não se esqueça de rodar o `go mod init` para inicializar o módulo do Go e o `go mod tidy` para instalar as dependências.

### 3. Criando o arquivo do SAM Template

Agora, vamos criar o arquivo `template.yaml` para configurar o deploy no AWS Lambda. O conteúdo do arquivo será o seguinte:

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 5
    MemorySize: 128
    Environment:
      Variables:
        LAMBDA_ENABLED: true
        
  Api:
    Cors:
      AllowMethods: "'GET,POST,OPTIONS'"
      AllowHeaders: "'Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token'"
      AllowOrigin: "'*'"

Resources:
  Fiber:
    Type: AWS::Serverless::Function
    Metadata:
      BuildMethod: go1.x
    Properties:
      CodeUri: .
      Handler: bootstrap
      Runtime: provided.al2023
      Events:
        GetResource:
          Type: Api
          Properties:
            Path: /{proxy+}
            Method: any

Outputs:
  ApiUrl:
    Description: "URL for the API endpoint"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/"
```

Este arquivo define a função Lambda e a configuração do API Gateway para expor sua API. Ele será utilizado para realizar o deploy da aplicação no AWS Lambda.

### 4. Repositório com o Código Completo
Este exemplo completo de aplicação pode ser encontrado no nosso repositório do GitHub aqui.
[Exemplo de aplicação com Golang e AWS Lambda](https://github.com/cmparrela/ddev-api-rest-golang-lambda)

### 5. Preparando para o Deploy com o AWS SAM
Para construir a aplicação, rode:
```bash
sam build
```

Para fazer o primeiro deploy, utilize o comando --guided para configurar o deploy interativamente:
```bash
sam deploy --guided
```

No final do processo, voce conseguirá ver a URL da sua API no terminal. Dessa forma

```bash
Key                 ApiUrl                                                                                              
Description         URL for the API endpoint                                                                            
Value               https://xq5vqngao8.execute-api.us-east-1.amazonaws.com/Prod/ 
```

Nos próximos deploys, você pode utilizar o comando:
```bash
sam build && sam deploy
```