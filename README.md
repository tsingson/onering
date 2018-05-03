# One Ring to Queue Them All

Well, no, it's not really just one ring, but a collection of lock-free (starvation/wait free even) ring buffers for different scenarios, so it's even better!
These queues don't use CAS operations to make them suitable for low latency/real-time environments and as a side effect of that,
they preserve total order of messages. As a reward for finding flaws/bugs in this, I offer 64bit of random numbers for each.

A couple of things in it were inspired by the very cool LMAX Disruptor, so thanks @mjpt777!
It's not anywhere near as intrusive and opinionated as the Disruptor though. It's not a framework and its main goal is to be (very) simple.

## Description

The package contains 4 related but different implementations
1. SPSC - Single Producer/Single Consumer - For cases when you just need to send messages from one thread/goroutine to another
2. MPSC - Multi-Producer/Single Consumer - When you need to send messages from many threads/goroutines into a single receiver
3. SPMC - Single Producer/Multi-Consumer - When you need to distribute messages from a single thread to many others
4. MPMC - Multi-Producer/Multi-Consumer - Many-to-Many


At the moment, all queues only support sending pointers (of any type). You can send non pointer types, but it will cause heap allocation. But you *can not* receive anything but pointers, don't even try, it will blow up.


## How to use it

### Common interface
    var queeue = onering.New{Size: N}.QueueType()
    queue.Put(*T)
    queue.Get(**T)
    queue.Consume(fn(onering.Iter, *T))
    queue.Close()

### Simplest case
```go
   import "github.com/pltr/onering"
   var queue = onering.New{Size: 8192}.MPMC()

   var src = int64(5)
   queue.Put(&src)
   queue.Close()
   var dst *int64
   // .Get expects a pointer to a pointer
   for queue.Get(&dst) {
       if *dst != src {
           panic("i don't know what's going on")
       }
   }
```
### Single consumer batching case
Batching consumption is strongly recommended in all single consumer cases, it's expected to have both higher throughput and lower latency

```go
    import "github.com/pltr/onering"
    var queue = onering.New{Size: 8192}.SPSC()

    var src = int64(5)
    queue.Put(&src)
    queue.Put(6) // WARNING: this will allocate memory on the heap and copy the value into it
    queue.Close()

    queue.Consume(func(it onering.Iter, dst *int64) {
        if *dst != src {
            panic("i don't know what's going on")
        }
        it.Stop()
    })
    // still one element left in the queue
    var dst *int64
    // Get will always expect a pointer to a pointer
    if !queue.Get(&dst) || *dst != 6 {
        panic("uh oh")
    }
    fmt.Println("Yay, batching works")
```
You can run both examples by `go run cmd/examples.go`


### Warnings
Currently this is highly experimental, so be careful. It also uses some dirty tricks to get around go's typesystem.
If you have a type mismatch between your sender and receiver or try to receive something unexpected, it will likely blow up.

### FAQ

* **Why four different implementations instead of just one (MPMC)?**
    _There are optimizations to be made in each case. They can have significant effect on performance._

* **Which one should I use?**
    _If you're not sure, MPMC will likely to be the safest choice._

* **I think I found a bug/something doesn't work as expectd**
    _Feel free to open an issue_

* **How fast is it?**
    _I haven't seen any faster, especially when it comes to latency distribution_

* **Did someone actually ask those questions above?**
    _No_

### Some benchmarks

Macbook pro 2.9 GHz Intel Core i7 (2017)

Rings:

    BenchmarkRingSPSC_Get-8             	100000000	        59.7 ns/op
    BenchmarkRingSPSC_GetNolock-8       	200000000	        20.8 ns/op
    BenchmarkRingSPSC_Consume-8         	300000000	        12.8 ns/op
    BenchmarkRingSPMC-8                 	100000000	        41.4 ns/op
    BenchmarkRingSPMC_NoLock1CPU-8      	200000000	        24.2 ns/op
    BenchmarkRingMPSC_GetLocked-8       	100000000	        58.6 ns/op
    BenchmarkRingMPSC_GetNoLock1CPU-8   	200000000	        27.2 ns/op
    BenchmarkRingMPSC_Batch-8           	200000000	        28.9 ns/op
    BenchmarkRingMPMC_Get-8             	100000000	        46.4 ns/op
    BenchmarkRingMPMC_Get1CPU-8         	100000000	        30.1 ns/op



Go channels:

    BenchmarkChanMPMC-8                 	50000000	        83.3 ns/op
    BenchmarkChan/SPSC-8                	100000000	        54.8 ns/op
    BenchmarkChan/SPSC_NoLock1CPU-8     	100000000	        45.6 ns/op
    BenchmarkChan/SPMC_Locked-8         	10000000	       356 ns/op
    BenchmarkChan/SPSC_NoLock-8         	100000000	        46.2 ns/op
    BenchmarkChan/SPMC-8                	10000000	       340 ns/op
    BenchmarkChan/SPMC_NoLock-8         	100000000	        46.3 ns/op

"NoLock" tests are tests without pinning goroutines via `runtime.LockOSThread()` - has nothing to do with mutexes or some such.

You can generally expect a 2-10x increase in performance, especially if you take advantage of batching and/or use a heavy multicore setup.
Do note that batching methods in them *do not* increase latency but, in fact, do the opposite.

Here's some (however flawed - it's hard to measure it precisely, so had to sample) latency disptribution:

    BenchmarkResponseTimesRing-8

    [Sample size: 2048 messages] 50: 18ns	75: 19ns	90: 22ns	99: 22ns	99.9: 22ns	99.99: 22ns	99.999: 22ns	99.9999: 22ns
    [Sample size: 2048 messages] 50: 17ns	75: 19ns	90: 21ns	99: 25ns	99.9: 33ns	99.99: 33ns	99.999: 33ns	99.9999: 33ns
    [Sample size: 2048 messages] 50: 15ns	75: 17ns	90: 18ns	99: 34ns	99.9: 46ns	99.99: 54ns	99.999: 77ns	99.9999: 77ns

    BenchmarkResponseTimesChannel-8
    [Sample size: 2048 messages] 50: 169ns	75: 170ns	90: 170ns	99: 170ns	99.9: 170ns	99.99: 170ns	99.999: 170ns	99.9999: 170ns
    [Sample size: 2048 messages] 50: 157ns	75: 205ns	90: 251ns	99: 352ns	99.9: 421ns	99.99: 421ns	99.999: 421ns	99.9999: 421ns
    [Sample size: 2048 messages] 50: 163ns	75: 222ns	90: 266ns	99: 317ns	99.9: 393ns	99.99: 448ns	99.999: 459ns	99.9999: 459ns

This is WIP, so the API is unstable at the moment - there are no guarantees about anything

Also: https://github.com/kellabyte/go-benchmarks/tree/master/queues
SPSC Get (bounded by time.Now() call)
![chart](https://camo.githubusercontent.com/553d9f8936ed5f298e1b3c0de1724d71b5c57cea/68747470733a2f2f692e696d6775722e636f6d2f78547a397645432e706e67
 "Queue Benchmark")

Special thanks to @kellabyte and @egonelbre
