---
# date: '2024-10-05T21:09:33-03:00'
title: 'Como fazer deploy de AWS Lambda Golang no GitHub Actions'
draft: true
showToc: false
categories:
- AWS Lambda
- Golang
- GitHub Actions
---

Neste post, vou dar continuidade à saga de conteúdos sobre AWS Lambda com Golang. Agora, vamos aprender como configurar um pipeline de CI/CD e realizar o deploy usando o GitHub Actions.

# GitHub Actions

O GitHub Actions é o serviço de CI/CD do GitHub que permite automatizar fluxos de trabalho de maneira simples e eficiente. Ele é altamente flexível e integrado ao GitHub, o que facilita sua utilização em projetos novos e existentes.

Embora a documentação oficial seja bastante completa ([confira aqui](https://docs.github.com/pt/actions)), neste post vou apresentar um exemplo prático e explicar como você pode começar a usar o GitHub Actions para fazer deploy de aplicações AWS Lambda escritas em Golang.

## Preço

O GitHub Actions é gratuito para repositórios públicos, porém para repositórios privados, existe um limite do que é possível usar gratuitamente, sendo eles:

- **2.000 minutos por mês** para execução de Actions.
- **500 MB de armazenamento** para artefatos gerados.

Caso ultrapasse a cota, você pode optar por planos pagos para aumentar os limites. (Os valores podem variar, então consulte a [página oficial de preços](https://github.com/features/actions#pricing) para mais detalhes.)

# Criando o arquivo de deploy

Eu recomendo que, primeiro, você leia esta etapa de criação do arquivo, entenda o que está sendo feito e, no final, encontrará o arquivo completo, que pode ser copiado e adaptado para sua necessidade.

Para começar, vamos criar uma pasta na raiz do projeto hospedado no GitHub chamada `.github`. Dentro dessa pasta, criaremos outra chamada `workflows`. Por fim, dentro dela, criaremos o arquivo do workflow para o deploy, com o nome que preferirmos. Aqui, vou usar `sam-pipeline.yml`. O resultado final será assim:

```bash
.github/workflows/sam-pipeline.ym
```

Vamos começar a escrever dentro desse arquivo. A lógica será a seguinte, sempre que uma release for publicada no GitHub, a pipeline será acionada para enviar o código para produção.

```yaml
name: Deploy to Production

on:
  release:
    types: [published]
...
```

Para evitar que múltiplas pipelines rodem em paralelo, utilizaremos o recurso de `concurrency`. Esse recurso agrupa as execuções de um mesmo grupo, nos vamos dar um nome para o grupo e configurando para cancelar execuções anteriores caso haja mais de uma rodando simultaneamente.

```yaml
...
concurrency:
  group: sam-pipeline-prod
  cancel-in-progress: true
...
```

Agora, vamos começar a criar os jobs. Os primeiros jobs são bastante simples e autoexplicativos então vou apenas colar o código:

```yaml
...
jobs:
  deploy-lambda:
    name: Deploy to AWS
    runs-on: ubuntu-latest
    environment:
      name:  'production'

    steps:
      - name: Checkout branch
        uses: actions/checkout@v4

      - name: Install Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22.2'
      
      - name: Download dependencies
        run: go mod tidy

      - name: Build with Go
        run: go build -o bootstrap main.go

      - name: Set up AWS SAM
        uses: aws-actions/setup-sam@v2
...
```

A próxima etapa é configurar as **AWS Credentials**. Agora, vamos adicionar as secrets no nosso repositório, que serão utilizadas na pipeline. No repositório, clique em **Settings → Actions secrets and variables → Actions** e adicione as chaves **AWS_ACCESS_KEY_ID** e **AWS_SECRET_ACCESS_KEY** de um usuário da AWS. 

O código para acessar as secrets no GitHub Action ficará assim:

```yaml
...
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-2

...
```

Para finalizar, vamos adicionar as etapas de **sam build** e **sam deploy**. Neste exemplo, usei o **SAM Template** da AWS e dei o nome do arquivo de produção como **template-prod.yaml**.

```yaml
...
      - name: Build with SAM
        run: sam build --template template-prod.yaml
      
      - name: Deploy with SAM
        run: sam deploy --no-confirm-changeset --no-fail-on-empty-changeset
...
```

# Arquivo completo

```yaml
# .github/workflows/sam-pipeline.ym
name: Deploy to Production

on:
  release:
    types: [published]

concurrency:
  group: sam-pipeline-prod
  cancel-in-progress: true
  
jobs:
  deploy-lambda:
    name: Deploy to AWS
    runs-on: ubuntu-latest
    environment:
      name:  'production'

    steps:
      - name: Checkout branch
        uses: actions/checkout@v4

      - name: Install Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22.2'
      
      - name: Download dependencies
        run: go mod tidy

      - name: Build with Go
        run: go build -o bootstrap main.go

      - name: Set up AWS SAM
        uses: aws-actions/setup-sam@v2
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-2

      - name: Build with SAM
        run: sam build --template template-prod.yaml
      
      - name: Deploy with SAM
        run: sam deploy --no-confirm-changeset --no-fail-on-empty-changeset
```