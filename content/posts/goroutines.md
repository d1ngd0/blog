+++
title = 'Goroutines'
date = 2024-02-03T13:51:42-06:00
draft = true
+++

I was having a conversation with an engineer about the difference between green threads and OS threads. I stated that Go uses green threads, and has an internal scheduler that juggles workloads accross OS threads. I was just regurgitating things I have heard in the past. In fact I believe I heard about green threads when suffering a rant from Gunter: "Real programmers use OS threads, I'd rather not have the language spoon feed me concurrency." You have fun with your self inflating CS masturbation Gunther; the rest of us are going to get some work done.

BUT I digress, as is often the case with these blog posts, Within this post I will walk through the implementation of go routines and what I find interesting.

## Greenthreads and the Scheduler

## Your Goroutines Bounce around

One assumption I made about the scheduler is it will keep a goroutine on the OS Thread, but was surprised to hear that wasn't the case. The scheduler has 3 levels of abstraction:

`G`: A goroutine, I hope you know what this is for.
`P`: Processors, not to be confused with CPUs. You can think of these as workers. `GOMAXPROCS` defines how many of these there will be.
`M`: An OS thread, the thing Gunther gets all excited about. If one of these isn't busy with a syscall it should be pulling work from the list of waiting `P`s

According to the design doc each `P` can end up jumping around:

> When an M is willing to start executing Go code, it must pop a P form the list. When an M ends executing Go code, it pushes the P to the list. So, when M executes Go code, it necessary has an associated P. This mechanism replaces sched.atomic (mcpu/mcpumax).

By default `GOMAXPROCS` is set to the number of cores on the machine. So it might be possible to keep things from jumping around. Maybe if you are careful you can keep things on the same thread?

> When a new G is created or an existing G becomes runnable, it is pushed onto a list of runnable goroutines of current P. When P finishes executing G, it first tries to pop a G from its own list of runnable goroutines; if the list is empty, P chooses a random victim (another P) and tries to steal half of runnable it's goroutines.

Well fine then! So yeah, your goroutine is going to bounce around across threads while running. At least according to the doc. We can verify this is the case with BPFtrace.

<sub>On a side note if you want to see some tea around this subject I stumbled on [this](https://github.com/golang/go/issues/23758)</sub>

# Sources or whatever

[Scheduler Design Doc](https://golang.org/s/go11sched)
