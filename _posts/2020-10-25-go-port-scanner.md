---
title: Building a Simple Port Scanner with GO
tags: GO
---

## Building a Simple Port-Scanner with Go

These days i wanted to make a new project with go, so i went to the [programming challenges](https://github.com/thinkbreak/programming-challenges) page and searched for something, the port
scanner caught my attention, so i decided to went for it.

----------

## Disclaimer

What we're gonna use build can't be used against servers which you don't have a permission to run. Read [this](https://nmap.org/book/legal-issues.html "nmap legal issues") to understand more

## Initial Version

So first we’ll define a function called `PortScan` which will accept a `server` parameter that will be the server which we’ll scan for it and return a list with the available ports. 
A port number is a 16-bit unsigned integer, thus ranging from 0 to 65535, but 0 is reserved and can’t be used, so we know the number of ports that we’ve to scan. Also we’ve to think 
about how we’re gonna check if the port is available, for this we can use the `net` package, he provides us the `Dial` function, which we can use to test the connection, but a better 
one would be the `DialTimeout` because it gives us the possibility to set a timeout to the connection. So let’s define our `PortScan()`

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

For this code to work we have to import the following packages:

`net`: Package for the `DialTimeout` function

`strconv`: To convert integers to strings and build our server variable

`time`: To create a *time.Second parameter

`os`: To get arguments from the command line (we’ll see where ahead)

`fmt`: To print results to the command line

Now for our main function

```go
func main() {

    fmt.Println("Cheking for available ports...")
        ports := PortScan(os.Args[1])

    fmt.Println("Ports available: " ,ports)

}
```

Now that we have our program, we want to build it, to use you can just run with `go run main.go <server>` where server is the ip address which you’ll want to check. Let’s use `time` command
to check for the time it takes to run:

```bash
┌─[nivaldogmelo@yggdrasil] - [/port-scanner]
└─[$] time go run port-scanner-no-goroutines.go localhost
Checking for available ports...
Ports available:  [4000 40031 42987 57621]

real    20.40s
user    8.74s
sys     12.14s

```

Note that it took a reasonable time, that’s because we check one port at a time, so let’s try to check for multiple ports at the same time

----------

## Using Goroutines

To increase the speed of our test we could check at multiple ports at the same time. For this we can use _go routines_, which are similar to _threads_ in languages like Java. If you don’t
know what a routine is i recommend reading the [Golang Bot](https://golangbot.com/learn-golang-series/) tutorial (sections 20–23) to have an idea about what we’re gonna handle.
The first thing we’re gonna do is import the `sync` package to deal with the goroutines. Now we’re gonna do some changes at the structure of the code. First we’re gonna make the `available`
variable a global one, because it will create be manipulated by multiple routines. Second it’s to create a struct to define a job that will be executed

```go
type Job struct {
        server string
        port   int
}

var available []int
var jobs = make(chan Job, 10)
```

The `jobs` variable keeps a buffered channel, which is a channel that will store a buffer of jobs to be executed, we defined the channel to have a length of 10, which means it can hold a 
maximum of 10 jobs at the same time, any other job will be blocked until some one of the jobs is terminated.

Now we’re gonna define our createWorkerPool(), which will create our workers to execute the job. Basically it’s here where we define how many concurrent jobs we’ll run at the same time.
To initiate a new goroutine we just need to execute our function with a `go` preceding it. We use the `wg.Add(1)` to add a new routine to execute our jobs. At the end the `wg.Wait()` 
is required to our main routine to wait for the subsequent routines to be completed before moving to the next step. The `go worker(&wg)` needs to use a pointer so that way it will use the
same `WaitGroup` created by the function, otherwise would create a new one and the tasks would be executed at another goroutines and our original wg group of routines would never be finished

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

Now we need to create a `worker()` to execute our job.

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
