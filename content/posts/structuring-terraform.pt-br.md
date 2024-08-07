+++
author = "nivaldo melo"
title = "Estruturando seu Terraform"
date = "2024-08-07"
description = "Um guia para como estruturar o terraform dentro do seu time"
tags = [ "golang", "terraform", ]
+++

## Estruturando seu Terraform (Em construção)

Como sabemos o Terraform é uma das ferramentas mais populares quando se trata de IAC declarativa, porém algo que sem pre me deixou um pouco perdido foi como estruturar meus  projetos. Não é difícil acharmos um tutorial de boas práticas de como devemos nomear, porém sempre fiquei um pouco perdido em questão de organização. Devo usar módulos? Workspaces? Até onde devo agregar os recursos em um único arquivo `.tf`?

A partir disso escrevi um modelo de estrutura para o código terraform, que engloba funcionalidades que na minha opinião são essenciais em um projeto. Esse modelo é composto por três pilares:

- **Módulos**: Utilizados para agregar serviços que funcionam como uma unidade (Mais detalhes a frente), dessa forma reduziremos a repetição de códigos.
- **Testes Unitários**: Para conseguirmos detectar falhas o mais cedo possível no desenvolvimento
- **Documentação**: Tentaremos deixa o processo de documentar o mais suave possível de forma que fique fácil para que outras pessoas da empresa consigam adotar os arquivos.

Vamos começar então explicando o motivo pela escolha desses pilares.

----------

### Módulos

Como muitos já devem saber, um módulo no Terraform é uma forma de agregarmos recursos que são configurado em conjunto com certa frequência, dessa forma conseguimos reaproveitar código com mais facilidade e fazer alterações em massa. Algo importante a ser considerado é que quando escrevendo módulos outras pessoas podem estar dependendo do seu código, por isso é importante levar em consideração antes de realizar uma alteração que possa levar a incompatibilidade, mas temos como contornar isso.

### Testes Unitários

Testes em infraestrutura de certa forma ainda são incomuns por conta do custo que isso pode gerar, porém se bem controlados podem facilitar bastante o desenvolvimento e manutenabilidade do seu código, podemos ter nos testes os cenários mais importantes aos quais os módulo deve cobrir, para evitar que uma alteração nova quebre algo existente. Outra vantagem é que tendo os cenários documentado nos testes não precisamos nos preocupar em realizarmos nós mesmos, o que pode levar a erros.

### Documentação

Por último temos a etapa da documentação, ter uma boa documentação facilita muito a compreensão e a adoção por parte dos times, porém se feita manualmente é muito fácil ter algum exemplo perdido, ou uma variável com valor errado em um readme, levando a falhas na hora que alguém tentar seguir a mesma. Por isso vamos tentar deixar o processo o mais automático possível, de forma que a doc esteja sempre atualizada com os valores no seu projeto.

### Onde queremos chegar

Por fim segue a estrutura de repositório onde queremos chegar, observe que temos alguns arquivos `go` pois é a linguagem que vamos utilizar para escrever os testes unitários

``` bash
terraform-modules-structure
├── examples/
├── go.mod
├── go.sum
├── modules/
├── README.md
└── test/
```

----------

## Escrevendo o código

Antes de começar vamos então definir dois tipos de serviços para subirmos, primeiro vamos focar em uma Cloud Function no GCP, em uma cloud Function podemos escolher alguns tipos de fontes para buscar o nosso código, para esse exemplo vamos buscar de um Bucket, então vamos ter a seguinte estrutura:

()()()()()()()

### Inicializando o Repositório

Essa parte começa bem simples, a princípio só precisamos iniciar um projeto golang, o que pode ser feito da seguinte forma

``` bash
❯ mkdir terraform-modules-structure && cd terraform-modules-structure
❯ go mod init terraform-modules-structure
```

Observe que nomeei meu projeto como `terraform-modules-structure`, mas fique livre pra nomear como quiser.

### Testes

Tentando seguir uma boa prática do mundo do desenvolvimento vamos começar pelos testes unitários. Para isso utilizaremos uma ferramenta chamada [Terratest](https://terratest.gruntwork.io/docs/getting-started/quick-start/) desenvolvida pelo pessoal do Gruntwork. Essa biblioteca possui vários módulos para lidar com diferentes backends (Google, AWS, Kubernetes, etc...), facilitando a adoção em diferentes projetos.

Agora vamos escrever nosso primeiro teste, primeiro vamos criar uma pasta para nossos testes e então o arquivo do mesmo, lembrando que em um projeto `go` testes costumam ter uma nomenclatura `*_test.go`.

```bash
❯ mkdir -p test/gcp
❯ touch test/gcp/cloud_function_test.go
```

Pensando agora no nosso serviço vamos analisar os requisitos que temos. Gostaríamos que o terraform que escrevemos tenha um retorno com sucesso, e que sejam criados uma instância de Cloud Function e um bucket.

Um ponto importante ao executar testes são o nome dos recursos e garantir que o mesmo seja destruído após a rotina. O primeiro ponto para evitar que o mesmo falhe somente por ter um nome repetiddo e o segundo para reduzir os gastos.

Como essas coisas são gerais em testes, vamos aproveitar para criar o arquivo `test/helpers.go` que terá as funções auxiliares para que possam ser reaproveitadas em multiplos testes.

```go
package test

import (
	"math/rand"
	"testing"
	"time"

	"github.com/gruntwork-io/terratest/modules/terraform"
)

// Teardown destrói os recursos criados pelo terraform apply, caso ocorra um erro, ele tenta novamente após 5 segundos
func Teardown(t *testing.T, terraformOptions *terraform.Options) {
	_, err := terraform.DestroyE(t, terraformOptions)
	for err != nil {
		time.Sleep(5 * time.Second)
		_, err = terraform.DestroyE(t, terraformOptions)
	}
}

// GenerateHash retorna uma string aleatória dado um tamanho.
func GenerateHash(size int) string {
	letterRunes := []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ")
	s := make([]rune, size)
	for i := range s {
		s[i] = letterRunes[rand.Intn(len(letterRunes))]
	}

	return string(s)
}
```

Agora o que precisamos fazer é escrever o teste de fato, para isso crie o arquivo `test/gcp/cloud_fucntion_test.go` e preencha com o seguinte

``` go
package gcp_test

import (
	"testing"

	// Observe que aqui é um import do meu projeto, o seu pode variar
	"github.com/nivaldogmelo/terraform-modules-structure/test"

	"github.com/gruntwork-io/terratest/modules/gcp"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestCloudFunction(t *testing.T) {
	// t.Parallel() is used to run the tests in parallel

	projectId := gcp.GetGoogleProjectIDFromEnvVar(t)

	testHash := test.GenerateHash(10)

	expectedFunctionName := "test-function" + testHash

	// Aqui definimos as configurações do Terraform, como o diretório onde estão os arquivos de configuração, variáveis e se o terratest deve realizar retry em caso de erro.
	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../../examples/gcp/cloud-function",

		VarFiles: []string{"inputs/varfiles.tfvars"},

		// Aqui definimos as variáveis que serão passadas para o Terraform.
		Vars: map[string]interface{}{
			"function_name": expectedFunctionName,
		},

		EnvVars: map[string]string{
			"GOOGLE_CLOUD_PROJECT": projectId,
		},
	})

	// Com o defer nós adiamos o uso do "terraform destroy" até o final do teste, assim garantimos que o ambiente será destruído mesmo que o teste falhe.
	defer test.Teardown(t, terraformOptions)

	// Aqui rodamos o comando "terraform init" e "terraform apply" para criar a infraestrutura.
	terraform.InitAndApply(t, terraformOptions)

	// Aqui verificamos se o bucket foi criado com sucesso.
	gcp.AssertStorageBucketExists(t, expectedFunctionName)
	// output := terraform.Output(t, terraformOptions, "hello_world")
	// assert.Equal(t, "Hello, World!", output)
}
```

Agora vamos executar os testes, para isso use

```bash
❯ go mod tidy
❯ go test ./...
```

Observe que o mesmo vai falhar, pois não configuramos nem o projeto nem os arquivos terraform. Agora vamos trabalhar nisso

### Exemplos

Eu diria que aqui é a parte mais importante do projeto, nessa parte vamos definir os exemplos de uso do módulo, pensando em como os usuários finais vão interagir com nossos módulos. Por isso é importante que os exemplos sejam claros, entreguem algo prático e sejam estáveis, pois caso contrário a adoção do projeto pode não ser simples.

Vamos pensar novamente no nosso exemplo, porém do ponto de vista de quem vai consumir o módulo. Precisamos criar uma Cloud Function no projeto GCP. A partir disso pensamos em quais parâmetros fazem sentido o usuário ter controle e quais não fazem tanta diferença.

Olhando o seguinte [exemplo](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloudfunctions2_function#example-usage---cloudfunctions2-basic) vamos começar proponto que o usuário possa ter controle nos seguintes pontos parâmetros:
- nome da funcão
- região
- descrição
- runtime
- entry_point
- max_instance
- memória
- timeout

O restante pode ser definido pelo próprio módulo, visto que não afeta tanto na entrega final

Então vamos começar a criar esses arquivos

``` bash
❯ mkdir -p examples/gcp/cloud-function-v2
❯ touch examples/gcp/cloud-function-v2/main.tf
```

No `main.tf` vamos preencher:

``` hcl
module "function" {
  source = "../../modules/gcp/cloud-function-v2"

  name        = "my-function"
  description = "My function"
  region      = "us-central1"

  runtime     = "nodejs16"
  entry_point = "helloGET"

  source_path = "./src"

  max_instances = 1
  memory        = 256
  timeout       = 60
}
```

Vamos criar uma função para conseguir fazer deploy no mesmo, para isso vamos usar o exemplo na própria doc do google, para encaixar o mesmo aqui basta usar

``` bash
❯ mkdir examples/gcp/cloud-function-v2/src
❯ touch examples/gcp/cloud-function-v2/src/index.js
❯ touch examples/gcp/cloud-function-v2/src/package.json
```

e preencher com o seguinte no `index.js`
``` javascript
const functions = require('@google-cloud/functions-framework');

// Register an HTTP function with the Functions Framework that will be executed
// when you make an HTTP request to the deployed function's endpoint.
functions.http('helloGET', (req, res) => {
  res.send('Hello World!');
});
```

e no `package.json`

``` json
{
  "name": "nodejs-docs-samples-functions-hello-world-get",
  "version": "0.0.1",
  "private": true,
  "license": "Apache-2.0",
  "author": "Google Inc.",
  "repository": {
	"type": "git",
	"url": "https://github.com/GoogleCloudPlatform/nodejs-docs-samples.git"
  },
  "engines": {
	"node": ">=16.0.0"
  },
  "scripts": {
	"test": "c8 mocha -p -j 2 test/*.test.js --timeout=6000 --exit"
  },
  "dependencies": {
	"@google-cloud/functions-framework": "^3.1.0"
  },
  "devDependencies": {
	"c8": "^8.0.0",
	"gaxios": "^6.0.0",
	"mocha": "^10.0.0",
	"wait-port": "^1.0.4"
  }
}
```

Pronto! Agora temos um exemplo de uso, falta fazer o mesmo funcionar.

### Modulos

Nessa parte é onde vai estar toda a lógica terraform por trás desse projeto, é esse o código que vai ser reaproveitado pelos demais times. O motivo pelo qual começamos pelos outros pontos e não por ele é que a ideia é termos módulos que funcionem de acordo com os times que usem, e não times que precisem se adequar para utilizar os módulos.

Vamos começar criando a pasta e os arquivos básicos pro módulo

``` bash
❯ mkdir -p modules/gcp/cloud-function-v2
❯ touch modules/gcp/cloud-function-v2/main.tf modules/gcp/cloud-function-v2/variables.tf
```

Agora vamos escrever a lógica terraform, no `main.tf` escreva:

``` hcl
resource "google_storage_bucket" "main" {
  // Aqui definimos o bucket que armazenará o código fonte da nossa função
  name                        = "${var.name}-gcf-source"
  location                    = "US"
  uniform_bucket_level_access = true
}

data "archive_file" "source" {
  // Com esse recurso nós zipamos o código fonte da nossa função para ser enviado ao bucket
  type        = "zip"
  source_dir  = var.source_path
  output_path = "function-source.zip"
}

resource "google_storage_bucket_object" "main" {
  // Aqui selecionamos o arquivo zipado e o enviamos para o bucket
  name         = data.archive_file.source.output_path
  bucket       = google_storage_bucket.main.name
  source       = data.archive_file.source.output_path
  content_type = "application/zip"

  // Dependemos do bucket e do arquivo para garantir que eles sejam criados antes da função
  depends_on = [ data.archive_file.source, google_storage_bucket.main ]
}

resource "google_cloudfunctions2_function" "main" {
  name        = var.name
  location    = var.location
  description = var.description

  build_config {
	runtime     = var.runtime
	entry_point = var.entry_point
	source {
	  storage_source {
	bucket = google_storage_bucket.main.name
	object = google_storage_bucket_object.main.name
	  }
	}
  }

  service_config {
	max_instance_count = var.max_instances
	available_memory   = var.memory
	timeout_seconds    = var.timeout
  }

  // Esperamos o objeto do bucket para garantir que o código fonte esteja disponível
  depends_on = [ google_storage_bucket_object.main ]
}
```

E vamos definir as variáveis no `variables.tf`

``` hcl
variable "name" {
  type        = string
  description = "The name of the function"
  nullable    = false
}

variable "description" {
  type        = string
  description = "The description of the function"
  default     = "My function"
}

variable "region" {
  type        = string
  description = "The region where the function will be deployed"
  default     = "us-central1"
}

variable "runtime" {
  type        = string
  description = "The runtime of the function"
  nullable    = false
}

variable "entry_point" {
  type        = string
  description = "The entry point of the function"
  nullable    = false
}

variable "source_path" {
  type        = string
  description = "The path to the source code of the function"
  default     = "./src"
}

variable "max_instances" {
  type        = number
  description = "The maximum number of instances that the function can have"
  default     = 1
}

variable "memory" {
  type        = number
  description = "The amount of memory in MB that the function can use"
  default     = 256
}

variable "timeout" {
  type        = number
  description = "The timeout in seconds for the function"
  default     = 60
}
```

### Doc

## Conclusão
