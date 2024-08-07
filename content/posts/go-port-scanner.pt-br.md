+++
author = "nivaldo melo"
title = "Construindo um Port Scanner com Golang"
date = "2020-10-25"
description = "Um guia para montar um port scanner com golang"
tags = [ "golang", ]
+++

## Construindo um Port Scanner com Golang

Esses dias eu quis fazer um projetinho com go, então visitei a [programming challenges](https://github.com/thinkbreak/programming-challenges) e busquei por algo, o port scanner chamou minha atenção então decidir pegar ele.

----------

## Disclaimer

O que vamos montar aqui não pode ser usado em servidores nos quais você não possui permissão para executar. Leia [essa](https://nmap.org/book/legal-issues.html "nmap legal issues") página para entender melhor

## Versão Inicial

Então primeiro vamos definir uma versão chamada `PortScan` que irá receber um parâmetro `servidor` o qual será o servidor que iremos escanear e retornar uma lista com as portas disponíveis no mesmo.

Um número de porta é um inteiro do tipo 16-bit unsigned, portanto seu range é de 0 a 65535, porém 0 é uma porta reservada e não pode ser usada, com isso já sabemos qual a quantidade de portas que teremos de verificar. Nós também precisamos pensar em como vamos verificar se uma porta está disponível, para isso usaremos o pacote `net`, ele nos fornece a função `Dial`, a qual podemos usar para testar a conexão, mas uma maneira melhor seria usar a `DialTimeout` pois nos dá a possibilidade de configurar um timeout para a conexão. Sem mais delongas vamos definir nossa `PortScan()`

```go
func PortScan(server string) []int {

        var available []int

        for i := 1; i <= 65535; i++ {
                ip := server + ":" + strconv.Itoa(i)

                _, err := net.DialTimeout("tcp", ip, time.Duration(300)*time.Millisecond)
                if err == nil {
                    available = append(available, i)
                }
        }

        return available

}
```

Para esse código funcionar precisamos importar os seguintes pacotes:

`net`: O pacote com a função `DialTimeout`

`strconv`: Para converter inteiros para strings e montar a nossa variável com o servidor

`time`: Para criar o parâmetro *time.Second

`os`: Para recuperar os argumentos da linha de comando (Veremos onde mais a frente)

`fmt`: Para exibir os resultados na linha de comando

Agora para nossa função principal

```go
func main() {

    fmt.Println("Cheking for available ports...")
        ports := PortScan(os.Args[1])

    fmt.Println("Ports available: " ,ports)

}
```

Agora que já temos nosso programa, vamos querer usá-lo, para usar isso você só precisa rodar `go run main.go <servidor>` onde o servidor é o endereço de IP que vc quer checar. Vamos usar o comando `time` para verificar o tempo que leva para ser executado:

```bash
┌─[nivaldogmelo@yggdrasil] - [/port-scanner]
└─[$] time go run port-scanner-no-goroutines.go localhost
Checking for available ports...
Ports available:  [4000 40031 42987 57621]

real    20.40s
user    8.74s
sys     12.14s

```

Observe que elvou um tempo razoável, isso se deve ao fato de que estamos checando uma porta de cada vez, vamos tentar acelerar a execução checando múltiplas portas ao mesmo tempo

----------

## Usando Goroutines

Para aumentar a velocidade do nosso teste podemos checar várias portas ao mesmo tempo. Para isso nós podemos usar _go routines_, o que são similares a _threads_ em linguagens como Java. Se você não sabe o que uma goroutine é recomendo a leitura do [Golang Bot](https://golangbot.com/learn-golang-series/) (em inglês) (seções 20-23) para ter uma ideia do que vamos lidar.

A primeira coisa que vamos fazer é importar o pacote `sync` para lidar com as goroutines. Agora vamos fazer algumas mudanças na estrutura do código. Primeiro vamos fazer da variável `available` uma variável global, pois será manipulada por múltiplas rotinas. Segundo é criar uma struct para definir o job que será executado.

```go
type Job struct {
        server string
        port   int
}

var available []int
var jobs = make(chan Job, 10)
```

A variável `jobs` mantem um canal bufferizado, que é um cnal que vai manter registro do buffer dos jobs que serão executados, nós definimos um canal de tamanho 10, o que significa que pode executar um total de 10 jobs ao mesmo tempo, qualquer outra execução será bloqueada até que os jobs em andamento sejam finalizados.

Agora vamos definir nossa função `createWorkerPool()`, a qual irá criar nossos workers para executar os jobs. Basicamente é aqui onde definiremos quantos jobs concorrentes queremos executar.

Para iniciar uma nova goroutine só precisamos executar nossa função com um `go` vindo antes. Nós usamos o `wg.Add(1)` para adicionar uma nova rotina para executar nossos jobs. Ao final o `wg.Wait()` é necessário para que nossa rotina principal espere pelas subsequentes serem finalizadas antes de ir para o próximo passo. O `go worker(&wg)` precisa usar um ponteiro de forma que use o mesmo `WaitGroup` criado pela função, caso contrário sempre iria iniciar um novo e as tarefas seriam executadas em outras goroutines e nosso grupo de espera original (wg) nunca seria finalizado

```go
func createWorkerPool(noOfWorkers int) {
        var wg sync.WaitGroup
        for i := 0; i < noOfWorkers; i++ {
                wg.Add(1)
                go worker(&wg)
        }
        wg.Wait()
}
```

Agora vamos criar a função `worker()` para executar nosso job.

```go
func worker(wg *sync.WaitGroup) {
        for job := range jobs{
                ip := job.server + ":" + strconv.Itoa(job.port)

                _, err := net.DialTimeout("tcp", ip, time.Duration(300)*time.Millisecond)
                if err == nil {
                        available = append(available, job.port)
                }
        }
        wg.Done()
}
```

Now we handle the ip assembly in the worker function, with parameters that we’ll receive from the `jobs` buffer, composed by variables with the `Job` struct type, which contains a server and a 
port. The net.DialTimeout will be executed as in our previous `PortScan()` and to finish the function we need to pass a `wg.Done()` , indicating the goroutine that the task is completed.

As we introduced these changes, we’ll have to change our original `PortScan()` . Now the function will be called with an additional parameter, a channel which will be written once all the jobs
are executed. For each port we’ll build a `Job` type variable and send it to our jobs list. At the end we’ll close the jobs channel since all jobs have been assigned and no other job will be
written to the channel. At the end we pass the `true` value to the `done` channel, to indicate we’ve finished the execution of all our goroutines.

```go
func PortScan(done chan bool, server string) {
        for i := 1; i <= 65535; i++ {
                job := Job{server, i}
                jobs <- job
        }
        close(jobs)
        done <- true
}
```

To end our implementation, we make the needed change at our `main()`

```go
func main() {
        fmt.Println("Cheking for available ports...")
        done := make(chan bool)
        go PortScan(done, os.Args[1])
        noOfWorkers := 10
        createWorkerPool(noOfWorkers)
        <-done

        fmt.Println("Ports available: " , available)

}
```

First we create our `done` channel, then we create a goroutine to run our `PortScan` getting the parameter that we’ll pass at the command line. Then we set a number of workers and 
create a worker pool of this size, for the sake of this demo we’ll go with 100 workers. Then we wait for the `done` channel to return a `true` value. So we only need to need to print
the available ports.

Ok so we’ve done a few changes on our code, but what have we achieved with this, so let’s use our `time` command to measure the performance:

```bash
┌─[nivaldogmelo@yggdrasil] - [/port-scanner]
└─[$] time go run port-scanner-goroutines.go localhost
Checking for available ports...
Ports available:  [4000 40031 42987 57621]

real    4.37s
user    4.93s
sys     5.21s
```

Now we've reached a smaller time

## Final considerations

When hiting a remote server be careful with the number of workers used, because some routers limit the number of concurrent threads, so some ports will be skipped

And that's it, i hope you guys enjoyed, if you have any questions you can send me an email or reach me through any of my social media accounts
