+++
title = "It's Go Time (bpftrace)"
date = 2024-03-19T05:39:53-05:00
draft = true
+++

No not that [go time](https://changelog.com/gotime), but they are worth listening to. I am instead going to discuss how to view a `time.Time` object in BPFtrace. I used to think it was just an `int64` like `time.Duration` is, but upon running into having to output a timestamp in BPFTrace I realized it was a bit more difficult. The structure of `time.Time` actually looks like this:

```go
type Time struct {
	// wall and ext encode the wall time seconds, wall time nanoseconds,
	// and optional monotonic clock reading in nanoseconds.
	//
	// From high to low bit position, wall encodes a 1-bit flag (hasMonotonic),
	// a 33-bit seconds field, and a 30-bit wall time nanoseconds field.
	// The nanoseconds field is in the range [0, 999999999].
	// If the hasMonotonic bit is 0, then the 33-bit field must be zero
	// and the full signed 64-bit wall seconds since Jan 1 year 1 is stored in ext.
	// If the hasMonotonic bit is 1, then the 33-bit field holds a 33-bit
	// unsigned wall seconds since Jan 1 year 1885, and ext holds a
	// signed 64-bit monotonic clock reading, nanoseconds since process start.
	wall uint64
	ext  int64

	// loc specifies the Location that should be used to
	// determine the minute, hour, month, day, and year
	// that correspond to this Time.
	// The nil location means UTC.
	// All UTC times are represented with loc==nil, never loc==&utcLoc.
	loc *Location
}
```

Go `time.Time` is actually a measurement of two different times: The Wall Clock, which is the time that you would see on a watch; and the monotonic clock reading, which is the time since the application started. The monotonic clock exists to protect your application from changes in time while measuring a duration. Let's look at the following code:

```go
start := time.Now()
... logic you are measuring ...
t := time.Now()
elapsed := t.Sub(start)
```

This would give you the correct answer most of the time, but say it was a very paticular saturday where we move the "Wall Clock" ahead or behind one hour; and lets say you are peticularly unlucky, and it happens right in the middle of your "... logic you are measuring ...". Now the duration of time would be off by -1 hour. Given all the software out there, this could easily crash more than a few programs in the middle of the night, and on a weekend no less! A Monotonic clock doesn't have to deal with this, since it **only** ever increases, and is a measure of time since the application started. 

We can write a quick program to output the monotonic clock as well. `time.Time` implements the `fmt.Stringer` function, and that includes the monotonic clock at the end. I wrote a simple program that loops over the time to output it as a string, and here is the output:

```
❯ ./go_string_date
2024-03-19 06:18:11.351047812 -0500 CDT m=+3.000143661
```

The wall clock is `2024-03-19 06:18:11.351047812 -0500` and the monotonic clock is `m=+3.000143661`. The [time package](https://pkg.go.dev/time#hdr-Monotonic_Clocks) goes into more detail if you are interested.

What kind of sucks is these two clocks are smashed together into `wall` and `ext`.

```go
// wall and ext encode the wall time seconds, wall time nanoseconds,
// and optional monotonic clock reading in nanoseconds.
```

So we are going to have to get creative to get the values out that we want.

## Getting Started

We can start by writing a program that outputs `time.Now` every second. I am going to wrap it in a function to make it easier to debug, since we can grab it as a function parameter.

```go
package main

import (
	"time"

	"github.com/davecgh/go-spew/spew"
)

func main() {
	spew.Config.DisableMethods = true
	spew.Config.MaxDepth = 1

	tick := time.NewTicker(time.Second)
	for {
		<-tick.C
		print_time(time.Now())
	}
}

//go:noinline
func print_time(time time.Time) {
	spew.FDump(os.Stdout, time)
}
```

Now we need to figure out where to pull the function arguments from. We can figure this out by dumping the assembly of the `print_time` function:

```
❯ objdump --disassemble=main.print_time gotime_bpftrace
000000000048db80 <main.print_time>:
...
  48db8a:	48 83 ec 48          	sub    $0x48,%rsp
  48db8e:	48 89 44 24 58       	mov    %rax,0x58(%rsp)
  48db93:	48 89 5c 24 60       	mov    %rbx,0x60(%rsp)
  48db98:	48 89 4c 24 68       	mov    %rcx,0x68(%rsp)
...
```

I have removed much of the output to focus in on what we actually care about, where we move the parameters out of the CPU registers and onto the functions stack. The `sub` is moving the stack pointer down, therefore making room for variables we are about to allocate there. Then we move `rax`, `rbx` and `rcx` to the stack. Since we are passing a value, and not a pointer, to `print_time` the whole object will be passed through function parameters. That means these three registers hold the contents of `time.Time`. referencing back we can see they are `wall uint64`, `ext int64` and `loc *location`.

In the code above `spew.Dump` will render the contents of our `time.Time`. We can verify we are getting the right values by writing a quick bpftrace to output what is in the cpu registers.

122
