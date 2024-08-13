+++
author = "nivaldo melo"
title = "Estruturando um projeto Terraform pt1"
date = "2024-08-13"
description = "Um guia para como estruturar o terraform dentro do seu time"
tags = [ "golang", "terraform", ]
+++

## Estruturando projetos Terraform Pt.1

Como sabemos o Terraform é uma das ferramentas mais populares quando se trata de Infraestrutura como código (IAC) declarativa. Porém algo que sempre me deixou um pouco perdido foi como estruturar meus projetos de maneira eficiente. Não é difícil achar um tutorial de boas práticas de como devemos nomear recursos, porém saber como organizar suas pastas pode ser um pouco mais desafiador. Devo usar módulos? Workspaces? Até onde devo agregar os recursos em um único arquivo `.tf`?

A partir disso escrevi um modelo de estrutura para o código terraform, que engloba funcionalidades que na minha opinião são essenciais em um projeto. Esse modelo é composto por quatro pilares:

- **Módulos**: Utilizados para agregar serviços que funcionam como uma unidade (Mais detalhes a frente), dessa forma reduziremos a repetição de códigos.
- **Exemplos**: Aqui serão definidos exemplos de uso para facilitar a adoção por outros membros do time.
- **Testes Unitários**: Para conseguirmos detectar falhas o mais cedo possível no desenvolvimento
- **Documentação**: Tentaremos deixar o processo de documentar o mais suave possível de forma que fique fácil para que outras pessoas da empresa consigam adotar os arquivos.

Vamos começar então explicando o motivo pela escolha desses pilares.

----------
### Módulos

Como muitos já sabem, um módulo no Terraform é uma forma de melhorar a reutilização de código, isso é feito através da agregação de recursos que são frequentemente criados juntos. Fazendo isso conseguimos centralizar a configuração em um único lugar, tornando alteraçóes em massa mais fáceis e eficientes. Algo importante a ser considerado é que quando escrevendo módulos outras pessoas podem estar dependendo do seu código, por isso é importante levar em consideração antes de realizar uma alteração que possa levar a incompatibilidade.

### Exemplos

Todos os exemplos devem ter uso prático, tornando fácil para os leitores verem como podem ser aplicados em seus próprios ambientes.

### Testes Unitários

Testes em infraestrutura costumam ser incomuns devido ao potencial de gastos, porém se feitos em um ambiente controlado podem facilitar bastante o desenvolvimento e manutenabilidade do seu código. Neles podemos ter descritos os cenários mais importantes aos quais os módulo devem cobrir, dessa forma evitando alterações que gerem incompatibilidade. Outra vantagem é que tendo os cenários documentado nos testes não precisamos nos preocupar com alguém testando manualmente todos os cenários, o que pode levar a erros.

### Documentação

Por último mas não menos importante, temos a etapa da documentação. Uma boa documentação torna mais fácil para outros times entenderem e adotarem seu produto. Porém se feita manualmente toda vez, é fácil perder controle das coisas, levando a uma documentação desatualizada e difícil de seguir. Por isso nós procuramos tornar o processo de documentar o mais suave possível, garantindo que esteja sempre de acordo com o código no projeto.

## Onde queremos chegar

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

Antes de começar vamos então definir o tipo de serviço que queremos subir. Para esse exemplo vamos criar uma Cloud Function no GCP. Em uma cloud Function podemos escolher alguns tipos de fontes para armazenar o nosso código fonte; para esse exemplo vamos utilizar um Bucket no GCS.

### Inicializando o Repositório

Essa parte é bem simples, a princípio só precisamos iniciar um projeto golang, o que pode ser feito da seguinte forma

``` bash
❯ mkdir terraform-modules-structure && cd terraform-modules-structure
❯ go mod init terraform-modules-structure
```

Observe que nomeei meu projeto como `terraform-modules-structure`, mas fique livre pra nomear como preferir.

### Exemplos

Eu diria que aqui é a parte mais importante do projeto, nessa parte vamos definir os exemplos de uso do módulo, pensando em como os usuários finais vão interagir com nossos módulos. Por isso é importante que os exemplos sejam claros, práticos e estáveis, pois caso contrário a adoção do projeto pode ser comprometida.

Pensando do ponto de vista do usuário final. Vamos decidir quais parâmetros fazem sentido controlarmos e quais não.

Olhando o seguinte [exemplo](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloudfunctions2_function#example-usage---cloudfunctions2-basic) vamos propor que o usuário possa ter controle nos seguintes parâmetros:
- nome da funcão
- região
- descrição
- runtime
- entry_point
- max_instance
- memória
- timeout

O restante pode ser definido no próprio módulo, incluíndo a integração com o bucket.

#### Arquivos Terraform

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

#### Inserindo código para função

Vamos criar uma função para conseguir fazer deploy da mesma, para isso vamos usar o exemplo na própria documentação oficial do google, para encaixar o mesmo na nossa estrutura basta fazer o seguinte:

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

Ótimo! Agora temos um caso de uso, só falta documentar o mesmo.

#### Documentação

Agora que temos um caso de uso, é interessante fornecer um README explicando como se utilizar, dessa forma tornando a adoção mais fácil. Para isso vamos utilizar uma ferramenta chamada [terraform-docs](https://terraform-docs.io/user-guide/introduction/). Após o instalar vamos configurar o seguinte:

Primeiro criamos o arquivo `examples/.terraform-docs.yml`, o qual vai fornecer uma estrutura base para ser utilizada em todos os exemplos.

```` yaml
formatter: "markdown table"
header-from: header.md # Indicamos de onde virá o conteúdo para servir de header do readme

content: |-
  {{ .Header }}

  ```hcl
  {{ include "main.tf" }}
  ```
````

Após isso criamos o arquivo `examples/gcp/cloud-function-v2/header.md`, aqui escrevemos um header específico para o nosso exemplo, nesse caso:

``` markdown
## Criando uma Cloud Function

Esse exemplo demonstra como criar uma Cloud Function no GCP via Terraform. O exemplo irá comprimir o código dentro da pasta `src/` e o salvar em um bucket no GCS.
```

Por fim basta executarmos o seguinte para gerar nosso README:

``` bash
❯ cd examples/gcp/cloud-function-v2
❯ terraform-docs -c ../../.terraform-docs.yml .
```

Com isso temos um caso de uso documentado e um processo para regerar com base no nosso código. Para automatizar o processo basta adicionar o mesmo no seu CI.

## Próximos passos

Na próxima parte vamos escrever um teste unitário tomando nosso exemplo como base.
