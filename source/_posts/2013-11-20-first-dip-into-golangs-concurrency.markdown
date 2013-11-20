---
layout: post
title: "First dip into Golang's concurrency"
date: 2013-11-20 11:53
comments: true
categories: golang, concurrency
---

I have been learning Google's [Go](http://golang.org) language lately.  The native support for concurrent programming is one of Go's major selling point.

Go has low-level primitives for concurrent programming such as [mutexes](http://golang.org/pkg/sync/#Mutex) and [atomic](http://golang.org/pkg/sync/atomic/), but also provides high-level language constructs for building concurrent programs via goroutines and channels.

Goroutines are functions executing in the same address space as other goroutines, like threads, but unlike threads, they communicate to each other via channels, not shared variables.

Channels provide a lock-free mechanism for goroutines to communicate.  To me, conceptually it feels a lot like a Unix socket: you can wait on it for data, or you can send data to it.  In Go, channels are also strongly and statically typed.

For me, the best way to learn something is to put it to practice.  I use one problem from [Project Euler](http://projecteuler.net).

    Find the sum of all prime numbers under 2 million

I wrote an Erlang version of this problem [before](/blog/2009/06/01/fast-and-elegant-way-to-sum-primes-in-a-gigantic-range/), but since then, Erlang kind of fell off my radar.  However, the problem and the concurrent solution is still relevant.

Test if a number is prime
=========================

I'll briefly go over primality test function, since it's not the focus of this blog post:

```go
func isPrime(n int) bool {
    if n == 1 || n == 2 {
        return true
    }

    if math.Mod(float64(n), 2) == 0 {
        return false
    }

    for i := 3.0; i <= math.Floor(math.Sqrt(float64(n))); i += 2.0 {
        if math.Mod(float64(n), i) == 0 {
            return false
        }
    }

    return true
}
```

I understand there are other faster primality tests but I opted for this basic algorithm for simplicity.

Non-concurrent version
======================

A naive way to solve this problem is to call `isPrime` on every number below 2 million, if it's a prime, add it to the tally.

```go
func sumPrimesUpto(n int) int {
    sum := 0
    for i := 1; i <= n; i++ {
        if isPrime(i) {
            sum += i
        }
    }

    return sum
}
```

Here's the main function:
```
func main() {
    upperBound, err := strconv.Atoi(os.Args[1])
    if err != nil {
        fmt.Println("Invalid argument.")
        os.Exit(1);
    }

    result := sumPrimesUpto(upperBound)
    fmt.Println(result)
}
```

Now run it and time it:
```
$ time go run sumprimes1.go 2000000
142913828923

real    0m27.032s
user    0m26.953s
sys     0m0.029s
```

Not too bad.  I remember when I ran this algorithm 4 years ago on my previous laptop (Core-2 Duo) I wasn't able to produce any result in a tolerable timeframe.  My current machine is a 3-year old Quad Core i7.


Concurrent version
==================

If you are on Linux and you open system monitor while the previous program was running, you can see that only one CPU was saturated and constantly running at near 100%, but all other cores are nearly idle.  Of course this is a huge waste of our computing resource.  `isPrime` function is what takes up the CPU load, and because we're running testing the primality of all 2 million numbers inside a single thread, all of them have to be tested one after the other.  This is not great.  Instead, because we have more than one CPU core, we can give the other cores chances to do some of the work for us.

If you were writing a Java or C++ program, you'd:
- make a variable for the sum
- loop from 1 to 2 million
- spawn a new thread to do the primality test
- inside the thread, if the primality test succeeds, lock the access to the `sum` variable, update `sum`, unlock

Programs like this have a higher complexity than it should.  It may not look like it's too complicated for this case, but synchronization using [locks](http://en.wikipedia.org/wiki/Lock_(computer_science)#Disadvantages) has inherent problems and is usually a source of bugs and defects.  Also, spawning as many threads as you can normally won't give you more throughput.  On the contrary, if you hand the OS more threads at once than the number of physical cores, context switching will happen and it will decrease your performance.

Go's approach is very similar to Erlang's in concept.  In Erlang, the actor processes can't share variables, but instead, they can send data to the other processes.  In Go, goroutines normally don't share variables, but they communicate via the use of channels.

Channels
--------

For this problem, we need to have the following channels:
- jobs: the outstanding jobs need to be performed.  Each job is a number whose primality needs to be tested.  It's a buffered channel whose size is the number of physical cores.
- results: the prime numbers that are already tested.  Buffered channel.  Can be as big as reasonable.
- done: whether all workers have finished their job. Also a buffered channel whose size is the number of physical cores.

Goroutines
----------

We need the following goroutines to:
- take the next number and put it in the `jobs` channel
- receive the next available job, run primality test, put the number on the `results` channel if succeeded, and signal the `done` channel.
- receive the signal from the `done` channel.  If no signals are received, we have done all the primality test.

Finally, we need to have a function to sum up all results.

Data structures
---------------

We want an abstraction of a `Job`.  In Go, that's a struct:

```go
type Job struct {
    n int
    results chan<-int
}
```

A job knows what number to test, and the results channel to which we can send the result.

A job also knows how to `Do`:
```go
func (job *Job) Do() {
    if isPrime(job.n) {
        job.results <- job.n
    }
}
```
In Go, a function with a receiver is practically a method on a struct and is able to be called with `receiver.method`.

Rewrite sumPrimesUpto
---------------------

Now, rewrite the `sumPrimesUpto` function:
```go
var workers = runtime.NumCPU()

func sumPrimesUpto(n int) int {
    jobs := make(chan Job, workers)
    results := make(chan int, n)
    done := make(chan struct{}, workers)

    go addJobs(jobs, results, n)

    for i := 0; i < workers; i++ {
        go doJobs(done, jobs)
    }

    go wait(done, results)

    return tally(results)
}
```

First, we need to know how many CPU cores the underlying platform knows about.  We only make the channel as big as the number of CPU cores.

Then, we make the channels.  One thing to note is that the `done` channel receives an empty `struct`, because we use that only for signaling.  We don't really care what value of the signal is.  We could define a surrogate type: `type Signal struct{}`, but an anonymous type will do just fine.

After that, we call `addJobs` as a goroutine.  The line `jobs <- Job{i, results}` will block if the channel is already full.
```go
func addJobs(jobs chan<-Job, results chan<-int, n int) {
    for i := 1; i <= n; i++ {
        jobs <- Job{i, results}
    }
    close(jobs)
}
```

In a separate goroutine, we take the jobs from the `jobs` channel and process them in `doJobs`:
```go
func doJobs(done chan<-struct{}, jobs <-chan Job) {
    for job := range jobs {
        job.Do()
    }
    done <- struct{}{}
}
```
We also signal the `done` channel when the job is done.  `struct{}{}` creates an instance of the anonymous type we use as the signal.

In another separate goroutine, we wait until there's no more signals on the `done` channel.  This means that we have finished processing all jobs:
```go
func wait(done <-chan struct{}, results chan int) {
    for i := 0; i < workers; i++ {
        <-done
    }
    close(results)
}
```
At this point, we can safely close the `results` channel as there won't be any new results coming in.

Finally, we can run `tally` on the results channel.
```go
func tally(results <-chan int) int {
    retval := 0
    for result := range results {
        retval += result
    }

    return retval
}
```

One thing worth mentioning is that even though the channels we made are all bi-directional channels, in the specific functions, we can make them more restrictive by making them send-only (chan<- Type) or receive-only (<-chan Type) according to their actual usage in the local function to avoid accidents.

Performance
-----------

So how does this concurrent version faire?

```
 $ time go run sumprimes.go 2000000
 CPUS=4
 142913828923

 real    0m12.534s
 user    0m44.289s
 sys     0m0.175s
```

On my Quad Core i7, it takes 12 seconds, almost twice as fast as the non-concurrent version!  And if you open System Monitor, you can see all 4 cores are running near 100%.


Conclusion
==========

So there's my first dip into Go's concurrency with an old problem. I like the concurrency primitives Go provides, even though it takes some getting used to.  Conceptually, goroutines are very similar to Erlang's actors.  Go has the advantage of a C-ish syntax that doesn't look like Prolog and it doesn't require a separate runtime as Erlang does. 
