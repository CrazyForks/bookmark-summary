Title: Gist of Go: Concurrency internals

URL Source: https://antonz.org/go-concurrency/internals/

Published Time: Sat, 06 Dec 2025 09:46:45 GMT

Markdown Content:
_This is a chapter from my book on [Go concurrency](https://antonz.org/go-concurrency), which teaches the topic from the ground up through interactive examples._

Here's where we started this book:

> Functions that run with `go` are called goroutines. The Go runtime juggles these goroutines and distributes them among operating system threads running on CPU cores. Compared to OS threads, goroutines are lightweight, so you can create hundreds or thousands of them.

That's generally correct, but it's a little too brief. In this chapter, we'll take a closer look at how goroutines work. We'll still use a simplified model, but it should help you understand how everything fits together.

[Concurrency](https://antonz.org/go-concurrency/internals/#concurrency) • [Goroutine scheduler](https://antonz.org/go-concurrency/internals/#goroutine-scheduler) • [GOMAXPROCS](https://antonz.org/go-concurrency/internals/#gomaxprocs) • [Concurrency primitives](https://antonz.org/go-concurrency/internals/#concurrency-primitives) • [Scheduler metrics](https://antonz.org/go-concurrency/internals/#scheduler-metrics) • [Profiling](https://antonz.org/go-concurrency/internals/#profiling) • [Tracing](https://antonz.org/go-concurrency/internals/#tracing) • [Keep it up](https://antonz.org/go-concurrency/internals/#keep-it-up)

Concurrency
-----------

At the hardware level, CPU _cores_ are responsible for running parallel tasks. If a processor has 4 cores, it can run 4 instructions at the same time — one on each core.

```
instr A     instr B     instr C     instr D
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│ Core 1  │ │ Core 2  │ │ Core 3  │ │ Core 4  │ CPU
└─────────┘ └─────────┘ └─────────┘ └─────────┘
```

At the operating system level, a _thread_ is the basic unit of execution. There are usually many more threads than CPU cores, so the operating system's scheduler decides which threads to run and which ones to pause. The scheduler keeps switching between threads to make sure each one gets a turn to run on a CPU, instead of waiting in line forever. This is how the operating system handles concurrency.

```
┌──────────┐              ┌──────────┐
│ Thread E │              │ Thread F │              OS
└──────────┘              └──────────┘
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Thread A │ │ Thread B │ │ Thread C │ │ Thread D │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
     │           │           │           │
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Core 1   │ │ Core 2   │ │ Core 3   │ │ Core 4   │ CPU
└──────────┘ └──────────┘ └──────────┘ └──────────┘
```

At the Go runtime level, a _goroutine_ is the basic unit of execution. The runtime scheduler runs a fixed number of OS threads, often one per CPU core. There can be many more goroutines than threads, so the scheduler decides which goroutines to run on the available threads and which ones to pause. The scheduler keeps switching between goroutines to make sure each one gets a turn to run on a thread, instead of waiting in line forever. This is how Go handles concurrency.

```
┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐
│ G15 ││ G16 ││ G17 ││ G18 ││ G19 ││ G20 │
└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘
┌─────┐      ┌─────┐      ┌─────┐      ┌─────┐
│ G11 │      │ G12 │      │ G13 │      │ G14 │      Go runtime
└─────┘      └─────┘      └─────┘      └─────┘
  │            │            │            │
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Thread A │ │ Thread B │ │ Thread C │ │ Thread D │ OS
└──────────┘ └──────────┘ └──────────┘ └──────────┘
```

The Go runtime scheduler doesn't decide which threads run on the CPU — that's the operating system scheduler's job. The Go runtime makes sure all goroutines run on the threads it manages, but the OS controls how and when those threads actually get CPU time.

Goroutine scheduler
-------------------

The scheduler's job is to run M goroutines on N operating system threads, where M can be much larger than N. Here's a simple way to do it:

1.   Put all goroutines in a queue.
2.   Take N goroutines from the queue and run them.
3.   If a running goroutine gets blocked (for example, waiting to read from a channel or waiting on a mutex), put it back in the queue and run the next goroutine from the queue.

Take goroutines G11-G14 and run them:

```
┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐
│ G15 ││ G16 ││ G17 ││ G18 ││ G19 ││ G20 │          queue
└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘
┌─────┐      ┌─────┐      ┌─────┐      ┌─────┐
│ G11 │      │ G12 │      │ G13 │      │ G14 │      running
└─────┘      └─────┘      └─────┘      └─────┘
  │            │            │            │
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Thread A │ │ Thread B │ │ Thread C │ │ Thread D │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
```

Goroutine G12 got blocked while reading from the channel. Put it back in the queue and replace it with G15:

```
┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐
│ G16 ││ G17 ││ G18 ││ G19 ││ G20 ││ G12 │          queue
└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘
┌─────┐      ┌─────┐      ┌─────┐      ┌─────┐
│ G11 │      │ G15 │      │ G13 │      │ G14 │      running
└─────┘      └─────┘      └─────┘      └─────┘
  │            │            │            │
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Thread A │ │ Thread B │ │ Thread C │ │ Thread D │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
```

But there are a few things to keep in mind.

**Starvation**

Let's say goroutines G11–G14 are running smoothly without getting blocked by mutexes or channels. Does that mean goroutines G15–G20 won't run at all and will just have to wait (_starve_) until one of G11–G14 finally finishes? That would be unfortunate.

That's why the scheduler checks each running goroutine roughly every 10 ms to decide if it's time to pause it and put it back in the queue. This approach is called preemptive scheduling: the scheduler can interrupt running goroutines when needed so others have a chance to run too.

**System calls**

The scheduler can manage a goroutine while it's running Go code. But what happens if a goroutine makes a system call, like reading from disk? In that case, the scheduler can't take the goroutine off the thread, and there's no way to know how long the system call will take. For example, if goroutines G11–G14 in our example spend a long time in system calls, all worker threads will be blocked, and the program will basically "freeze".

To solve this problem, the scheduler starts new threads if the existing ones get blocked in a system call. For example, here's what happens if G11 and G12 make system calls:

```
┌─────┐┌─────┐┌─────┐┌─────┐
│ G17 ││ G18 ││ G19 ││ G20 │                        queue
└─────┘└─────┘└─────┘└─────┘

┌─────┐      ┌─────┐      ┌─────┐      ┌─────┐
│ G15 │      │ G16 │      │ G13 │      │ G14 │      running
└─────┘      └─────┘      └─────┘      └─────┘
  │            │            │            │
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Thread E │ │ Thread F │ │ Thread C │ │ Thread D │
└──────────┘ └──────────┘ └──────────┘ └──────────┘

┌─────┐      ┌─────┐
│ G11 │      │ G12 │                                syscalls
└─────┘      └─────┘
  │            │
┌──────────┐ ┌──────────┐
│ Thread A │ │ Thread B │
└──────────┘ └──────────┘
```

Here, the scheduler started two new threads, E and F, and assigned goroutines G15 and G16 from the queue to these threads.

When G11 and G12 finish their system calls, the scheduler will stop or terminate the extra threads (E and F) and keep running the goroutines on four threads: A-B-C-D.

This is a simplified model of how the goroutine scheduler works in Go. If you want to learn more, I recommend watching the talk by Dmitry Vyukov, one of the scheduler's developers: [Go scheduler: Implementing language with lightweight concurrency](https://www.youtube.com/watch?v=-K11rY57K7k) ([video](https://www.youtube.com/watch?v=-K11rY57K7k), [slides](https://assets.ctfassets.net/oxjq45e8ilak/48lwQdnyDJr2O64KUsUB5V/5d8343da0119045c4b26eb65a83e786f/100545_516729073_DMITRII_VIUKOV_Go_scheduler_Implementing_language_with_lightweight_concurrency.pdf))

GOMAXPROCS
----------

We said that the scheduler uses N threads to run goroutines. In the Go runtime, the value of N is set by a parameter called `GOMAXPROCS`.

The `GOMAXPROCS` runtime setting controls the maximum number of operating system threads the Go scheduler can use to execute goroutines concurrently (not counting the goroutines running syscalls). It defaults to the value of `runtime.NumCPU`, which is the number of logical CPUs on the machine.

> Strictly speaking, `runtime.NumCPU` is either the total number of logical CPUs or the number allowed by the CPU affinity mask, whichever is lower. This can be adjusted by the CPU quota, as explained below.

For example, on my 8-core laptop, the default value of `GOMAXPROCS` is also 8:

```
maxProcs := runtime.GOMAXPROCS(0) // returns the current value
fmt.Println("NumCPU:", runtime.NumCPU())
fmt.Println("GOMAXPROCS:", maxProcs)
```

```
NumCPU: 8
GOMAXPROCS: 8
```

You can change `GOMAXPROCS` by setting `GOMAXPROCS` environment variable or calling `runtime.GOMAXPROCS()`:

```
// Get the default value.
fmt.Println("GOMAXPROCS default:", runtime.GOMAXPROCS(0))

// Change the value.
runtime.GOMAXPROCS(1)
fmt.Println("GOMAXPROCS custom:", runtime.GOMAXPROCS(0))
```

```
GOMAXPROCS default: 8
GOMAXPROCS custom: 1
```

You can also undo the manual changes and go back to the default value set by the runtime. To do this, use the `runtime.SetDefaultGOMAXPROCS` function (Go 1.25+):

```
GOMAXPROCS=2 go run nproc.go
```

```
// Using the environment variable.
fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0))

// Using the manual setting.
runtime.GOMAXPROCS(4)
fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0))

// Back to the default value.
runtime.SetDefaultGOMAXPROCS()
fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0))
```

```
GOMAXPROCS: 2
GOMAXPROCS: 4
GOMAXPROCS: 8
```

### CPU quota

Go programs often run in containers, like those managed by Docker or Kubernetes. These systems let you limit the CPU resources for a container using a Linux feature called _cgroups_.

A cgroup (control group) in Linux lets you group processes together and control how much CPU, memory, and network I/O they can use by setting limits and priorities.

For example, here's how you can limit a Docker container to use only four CPUs:

```
docker run --cpus=4 golang:1.24-alpine go run /app/nproc.go
```

```
// /app/nproc.go
maxProcs := runtime.GOMAXPROCS(0) // returns the current value
fmt.Println("NumCPU:", runtime.NumCPU())
fmt.Println("GOMAXPROCS:", maxProcs)
```

Before version 1.25, the Go runtime didn't consider the CPU quota when setting the `GOMAXPROCS` value. No matter how you limited CPU resources, `GOMAXPROCS` was always set to the number of logical CPUs on the host machine:

```
docker run --cpus=4 golang:1.24-alpine go run /app/nproc.go
```

```
NumCPU: 8
GOMAXPROCS: 8
```

Starting with version 1.25, the Go runtime respects the CPU quota:

```
docker run --cpus=4 golang:1.25-alpine go run /app/nproc.go
```

```
NumCPU: 8
GOMAXPROCS: 4
```

So, the default `GOMAXPROCS` value is set to either the number of logical CPUs or the CPU limit enforced by cgroup settings for the process, whichever is lower.

**Note on CPU limits**

Cgroups actually offer not just one, but two ways to limit CPU resources:

*   CPU quota — the maximum CPU time the cgroup may use within some period window.
*   CPU shares — relative CPU priorities given to the kernel scheduler.

Docker's `--cpus` and `--cpu-period`/`--cpu-quota` set the quota, while `--cpu-shares` sets the shares.

Kubernetes' CPU limit sets the quota, while CPU request sets the shares.

Go's runtime `GOMAXPROCS` only takes the CPU quota into account, not the shares.

Fractional CPU limits are rounded up:

```
docker run --cpus=2.3 golang:1.25-alpine go run /app/nproc.go
```

```
NumCPU: 8
GOMAXPROCS: 3
```

On a machine with multiple CPUs, the minimum default value for `GOMAXPROCS` is 2, even if the CPU limit is set lower:

```
docker run --cpus=1 golang:1.25-alpine go run /app/nproc.go
```

```
NumCPU: 8
GOMAXPROCS: 2
```

The Go runtime automatically updates `GOMAXPROCS` if the CPU limit changes. It happens up to once per second (less frequently if the application is idle).

Concurrency primitives
----------------------

Let's take a quick look at the three main concurrency tools for Go: goroutines, channels, and select.

### Goroutine

A goroutine is implemented as a pointer to a `runtime.g` structure. Here's what it looks like:

```
// runtime/runtime2.go
type g struct {
    atomicstatus atomic.Uint32 // goroutine status
    stack        stack         // goroutine stack
    m            *m            // thread that runs the goroutine
    // ...
}
```

The `g` structure has many fields, but most of its memory is taken up by the stack, which holds the goroutine's local variables. By default, each stack gets 2 KB of memory, and it grows if needed.

Because goroutines use very little memory, they're much more efficient than operating system threads, which usually need about 1 MB each. Also, switching between goroutines is very fast because it's handled by Go's scheduler and doesn't involve the operating system's kernel (unlike switching between threads managed by the OS). This lets Go run hundreds of thousands, or even millions, of goroutines on a single machine.

### Channel

A channel is implemented as a pointer to a `runtime.hchan` structure. Here's what it looks like:

```
// runtime/chan.go
type hchan struct {
    // channel buffer
    qcount   uint           // number of items in the buffer
    dataqsiz uint           // buffer array size
    buf      unsafe.Pointer // pointer to the buffer array

    // closed channel flag
    closed uint32

    // queues of goroutines waiting to receive and send
    recvq waitq // waiting to receive from the channel
    sendq waitq // waiting to send to the channel

    // protects the channel state
    lock mutex

    // ...
}
```

The buffer array (`buf`) has a fixed size (`dataqsiz`, which you can get with the `cap()` builtin). It's created when you make a buffered channel. The number of items in the channel (`qcount`, which you can get with the `len()` builtin) increases when you send to the channel and decreases when you receive from it.

The `close()` builtin sets the `closed` field to 1.

Sending an item to an unbuffered channel, or to a buffered channel that's already full, puts the goroutine into the `sendq` queue. Receiving from an empty channel puts the goroutine into the `recvq` queue.

### Select

The select logic is implemented in the `runtime.selectgo` function. It's a huge function that takes a list of select cases and (very simply put) works as follows:

*   Go through the cases and check if the matching channels are ready to send or receive.
*   If several cases are ready, choose one at random (to prevent starvation, where some cases are always chosen and others are never chosen).
*   Once a case is selected, perform the send or receive operation on the matching channel.
*   If there is a default case and no other cases are ready, pick the default.
*   If no cases are ready, block the goroutine and add it to the channel queue for each case.

**✎ Exercise: Runtime simulator**

Practice is crucial in turning abstract knowledge into skills, making theory alone insufficient. The full version of the book contains a lot of exercises — that's why I recommend [getting it](https://antonz.gumroad.com/l/go-concurrency).

If you are okay with just theory for now, let's continue.

Scheduler metrics
-----------------

Metrics show how the Go runtime is performing, like how much heap memory it uses or how long garbage collection pauses take. Each metric has a unique name (for example, `/sched/gomaxprocs:threads`) and a value, which can be a number or a histogram.

We use the `runtime/metrics` package to work with metrics.

List all available metrics with descriptions:

```
func main() {
    descs := metrics.All()
    for _, d := range descs {
        fmt.Printf("Name: %s\n", d.Name)
        fmt.Printf("Description: %s\n", d.Description)
        fmt.Printf("Kind: %s\n", kindToString(d.Kind))
        fmt.Println()
    }
}

func kindToString(k metrics.ValueKind) string {
    switch k {
    case metrics.KindUint64:
        return "KindUint64"
    case metrics.KindFloat64:
        return "KindFloat64"
    case metrics.KindFloat64Histogram:
        return "KindFloat64Histogram"
    case metrics.KindBad:
        return "KindBad"
    default:
        return "Unknown"
    }
}
```

```
Name: /cgo/go-to-c-calls:calls
Description: Count of calls made from Go to C by the current process.
Kind: KindUint64

Name: /cpu/classes/gc/mark/assist:cpu-seconds
Description: Estimated total CPU time goroutines spent performing GC
tasks to assist the GC and prevent it from falling behind the application.
This metric is an overestimate, and not directly comparable to system
CPU time measurements. Compare only with other /cpu/classes metrics.
Kind: KindFloat64
...
```

Get the value of a specific metric:

```
samples := []metrics.Sample{
    {Name: "/sched/gomaxprocs:threads"},
    {Name: "/sched/goroutines:goroutines"},
}
metrics.Read(samples)

for _, s := range samples {
    // Assumes the value is a uint64. Check the metric description
    // or use s.Value.Kind() if you're not sure.
    fmt.Printf("%s: %v\n", s.Name, s.Value.Uint64())
}
```

```
/sched/gomaxprocs:threads: 8
/sched/goroutines:goroutines: 1
```

Here are some goroutine-related metrics:

`/sched/goroutines-created:goroutines`

*   Count of goroutines created since program start (Go 1.26+).

`/sched/goroutines:goroutines`

*   Count of live goroutines (created but not finished yet).
*   An increase in this metric may indicate a goroutine leak.

`/sched/goroutines/not-in-go:goroutines`

*   Approximate count of goroutines running or blocked in a system call or cgo call (Go 1.26+).
*   An increase in this metric may indicate problems with such calls.

`/sched/goroutines/runnable:goroutines`

*   Approximate count of goroutines ready to execute, but not executing (Go 1.26+).
*   An increase in this metric may mean the system is overloaded and the CPU can't keep up with the growing number of goroutines.

`/sched/goroutines/running:goroutines`

*   Approximate count of goroutines executing (Go 1.26+).
*   Always less than or equal to `/sched/gomaxprocs:threads`.

`/sched/goroutines/waiting:goroutines`

*   Approximate count of goroutines waiting on a resource — I/O or sync primitives (Go 1.26+).
*   An increase in this metric may indicate issues with mutex locks, other synchronization blocks, or I/O issues.

`/sched/threads/total:threads`

*   The current count of live threads that are owned by the runtime (Go 1.26+).

`/sched/gomaxprocs:threads`

*   The current `runtime.GOMAXPROCS` setting — the maximum number of operating system threads the scheduler can use to execute goroutines concurrently.

In real projects, runtime metrics are usually exported automatically with client libraries for Prometheus, OpenTelemetry, or other observability tools. Here's an example for Prometheus:

```
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
    // Export runtime/metrics in Prometheus format at the /metrics endpoint.
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe("localhost:2112", nil)
}
```

The exported metrics are then collected by Prometheus, visualized, and used to set up alerts.

Profiling
---------

Profiling helps you understand exactly what the program is doing, what resources it uses, and where in the code this happens. Profiling is often not recommended in production because it's a "heavy" process that can slow things down. But that's not the case with Go.

Go's profiler is designed for production use. It uses sampling, so it doesn't track every single operation. Instead, it takes quick snapshots of the runtime every 10 ms and puts them together to give you a full picture.

Go supports the following profiles:

*   **CPU**. Shows how much CPU time each function uses. Use it to find performance bottlenecks if your program is running slowly because of CPU-heavy tasks.
*   **Heap**. Shows the heap memory currently used by each function. Use it to detect memory leaks or excessive memory usage.
*   **Allocs**. Shows which functions have used heap memory since the profiler started (not just currently). Use it to optimize garbage collection or reduce allocations that impact performance.
*   **Goroutine**. Shows the stack traces of all current goroutines. Use it to get an overview of what the program is doing.
*   **Block**. Shows where goroutines block waiting on synchronization primitives like channels, mutexes and wait groups. Use it to identify synchronization bottlenecks and issues in data exchange between goroutines. Disabled by default.
*   **Mutex**. Shows lock contentions on mutexes and internal runtime locks. Use it to find "problematic" mutexes that goroutines are frequently waiting for. Disabled by default.

The easiest way to add a profiler to your app is by using the `net/http/pprof` package. When you import it, it automatically registers HTTP handlers for collecting profiles:

```
package main

import (
    "net/http"
    _ "net/http/pprof"
    "sync"
)

func main() {
    // Enable block and mutexe profiles.
    runtime.SetBlockProfileRate(1)
    runtime.SetMutexProfileFraction(1)
    // Start an HTTP server on localhost.
    // Profiler HTTP handlers are automatically
    // registered when you import "net/http/pprof".
    http.ListenAndServe("localhost:6060", nil)
}
```

Or you can register profiler handlers manually:

```
var wg sync.WaitGroup

wg.Go(func() {
    // Application server running on port 8080.
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Hello, World!"))
    })
    log.Println("Starting hello server on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
})

wg.Go(func() {
    // Profiling server running on localhost on port 6060.
    runtime.SetBlockProfileRate(1)
    runtime.SetMutexProfileFraction(1)

    mux := http.NewServeMux()
    mux.HandleFunc("/debug/pprof/", pprof.Index)
    mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
    mux.HandleFunc("/debug/pprof/trace", pprof.Trace)
    log.Println("Starting pprof server on :6060")
    log.Fatal(http.ListenAndServe("localhost:6060", mux))
})

wg.Wait()
```

After that, you can start profiling with a specific profile by running the `go tool pprof` command with the matching URL, or just open that URL in your browser:

```
go tool pprof -proto \
  "http://localhost:6060/debug/pprof/profile?seconds=N" > cpu.pprof

go tool pprof -proto \
  http://localhost:6060/debug/pprof/heap > heap.pprof

go tool pprof -proto \
  http://localhost:6060/debug/pprof/allocs > allocs.pprof

go tool pprof -proto \
  http://localhost:6060/debug/pprof/goroutine > goroutine.pprof

go tool pprof -proto \
  http://localhost:6060/debug/pprof/block > block.pprof

go tool pprof -proto \
  http://localhost:6060/debug/pprof/mutex > mutex.pprof
```

For the CPU profile, you can choose how long the profiler runs (the default is 30 seconds). Other profiles are taken instantly.

After running the profiler, you'll get a binary file that you can open in the browser using the same `go tool pprof` utility. For example:

```
go tool pprof -http=localhost:8080 cpu.pprof
```

The pprof web interface lets you view the same profile in different ways. My personal favorites are the _flame graph_, which clearly shows the call hierarchy and resource usage, and the _source_ view, which shows the exact lines of code.

![Image 1: Flame graph view](https://antonz.org/go-concurrency/internals/pprof-flame-graph.png)

The flame graph view shows the call hierarchy and resource usage.

![Image 2: Source view](https://antonz.org/go-concurrency/internals/pprof-source.png)

The source view shows the exact lines of code.

You can also profile manually. To collect a CPU profile, use `StartCPUProfile` and `StopCPUProfile`:

```
func main() {
    // Start profiling and stop it when main exits.
    // Ignore errors for simplicity.
    file, _ := os.Create("cpu.prof")
    defer file.Close()
    pprof.StartCPUProfile(file)
    defer pprof.StopCPUProfile()

    // The rest of the program code.
    // ...
}
```

To collect other profiles, use `Lookup`:

```
// profile collects a profile with the given name.
func profile(name string) {
    // Ignore errors for simplicity.
    file, _ := os.Create(name + ".prof")
    defer file.Close()
    p := pprof.Lookup(name)
    if p != nil {
        p.WriteTo(file, 0)
    }
}

func main() {
    runtime.SetBlockProfileRate(1)
    runtime.SetMutexProfileFraction(1)

    // ...
    profile("heap")
    profile("allocs")
    // ...
}
```

Profiling is a broad topic, and we've only touched the surface. To learn more, start with these articles:

*   [Profiling Go Programs](https://go.dev/blog/pprof)
*   [Diagnostics](https://go.dev/doc/diagnostics)

Tracing
-------

Tracing records certain types of events while the program is running, mainly those related to concurrency and memory:

*   goroutine creation and state changes;
*   system calls;
*   garbage collection;
*   heap size changes;
*   and more.

If you enabled the profiling server as described earlier, you can collect a trace using this URL:

```
http://localhost:6060/debug/pprof/trace?seconds=N
```

Trace files can be quite large, so it's better to use a small N value.

After tracing is complete, you'll get a binary file that you can open in the browser using the `go tool trace` utility:

```
go tool trace -http=localhost:6060 trace.out
```

In the trace web interface, you'll see each goroutine's "lifecycle" on its own line. You can zoom in and out of the trace with the W and S keys, and you can click on any event to see more details:

![Image 3: Trace web interface](https://antonz.org/go-concurrency/internals/pprof-trace.png)
You can also collect a trace manually:

```
func main() {
    // Start tracing and stop it when main exits.
    // Ignore errors for simplicity.
    file, _ := os.Create("trace.out")
    defer file.Close()
    trace.Start(file)
    defer trace.Stop()

    // The rest of the program code.
    // ...
}
```

### Flight recorder

Flight recording is a tracing technique that collects execution data, such as function calls and memory allocations, within a sliding window that's limited by size or duration. It helps to record traces of interesting program behavior, even if you don't know in advance when it will happen.

The `trace.FlightRecorder` type (Go 1.25+) implements a flight recorder in Go. It tracks a moving window over the execution trace produced by the runtime, always containing the most recent trace data.

Here's an example of how you might use it.

First, configure the sliding window:

```
// Configure the flight recorder to keep
// at least 5 seconds of trace data,
// with a maximum buffer size of 3MB.
// Both of these are hints, not strict limits.
cfg := trace.FlightRecorderConfig{
    MinAge:   5 * time.Second,
    MaxBytes: 3 << 20, // 3MB
}
```

Then create the recorder and start it:

```
// Create and start the flight recorder.
rec := trace.NewFlightRecorder(cfg)
rec.Start()
defer rec.Stop()
```

Continue with the application code as usual:

```
// Simulate some workload.
done := make(chan struct{})
go func() {
    defer close(done)
    const n = 1 << 20
    var s []int
    for range n {
        s = append(s, rand.IntN(n))
    }
    fmt.Printf("done filling slice of %d elements\n", len(s))
}()
<-done
```

Finally, save the trace snapshot to a file when an important event occurs:

```
// Save the trace snapshot to a file.
file, _ := os.Create("/tmp/trace.out")
defer file.Close()
n, _ := rec.WriteTo(file)
fmt.Printf("wrote %dB to trace file\n", n)
```

```
done filling slice of 1048576 elements
wrote 8441B to trace file
```

Use `go tool trace` to view the trace in the browser:

```
go tool trace -http=localhost:6060 /tmp/trace.out
```

**✎ Exercise: Comparing blocks**

Practice is crucial in turning abstract knowledge into skills, making theory alone insufficient. The full version of the book contains a lot of exercises — that's why I recommend [getting it](https://antonz.gumroad.com/l/go-concurrency).

If you are okay with just theory for now, let's continue.

Keep it up
----------

Now you can see how challenging the Go scheduler's job is. Fortunately, most of the time you don't need to worry about how it works behind the scenes — sticking to goroutines, channels, select, and other synchronization primitives is usually enough.

This is the final chapter of my "Gist of Go: Concurrency" book. I invite you to read it — the book is an easy-to-understand, interactive guide to concurrency programming in Go.

[Pre-order for $10](https://antonz.gumroad.com/l/go-concurrency) or [read online](https://antonz.org/go-concurrency/)

[★Subscribe](https://antonz.org/subscribe/) to keep up with new posts.