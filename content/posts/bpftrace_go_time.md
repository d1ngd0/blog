+++
title = 'Grabbing a Go Time with BPFTrace'
date = 2024-10-06T08:45:05-06:00
draft = false
+++

Recently I found myself debugging an application where I needed to understand what value a `time.Time` object had. What I thought was going to be a trivial experience turned into a multi day confusing conundrum which left me confuddled and cranky. Alright, that is already too much alliteration for the first paragraph of the post. Long story short, I gave up and found a duration which gave me the context I needed to understand what was going on.

While finding a new solution to get around my ignorance was the right call, it always bugged me that I couldn't figure it out. Over the past 3 days this cantankerous quandary was completed (sorry). I will say I don't totally love the solution, but at least my ignorance isn't dictating the outcome anymore. More my lack of motivation to keep working on this story.

## The Setup

Alright, let's get to the heart of it. We want to create a Go application which passes a `time.Time` object to a method. It should do this at a regular interval so we can see it easily with BPFTrace.

```go
package main

import (
	"fmt"
	"strings"
	"time"

	"github.com/davecgh/go-spew/spew"
)

func main() {
	// This sets up spew so it doesn't display too much data
	spew.Config.DisableMethods = true
	spew.Config.MaxDepth = 1
	spew.Config.Indent = ""

	// create a new ticker which will create a time every
	// second
	tick := time.NewTicker(time.Second)
	for {
		t := <-tick.C
		// call out to print time
		print_time(t)
	}
}

//go:noinline
func print_time(time time.Time) {
	fmt.Println(strings.ReplaceAll(spew.Sdump(time), "\n", " "))
}

// the go:noiline stops the compiler from removing the print_time
// function all together. Compilers often "inline" functions for
// a performance increase. We need to retain this call in the
// binary so we can hook into it.
```

Let's then pair this with a BPFTrace program so we can see every time the function is called.

```
#!/usr/bin/bpftrace

/*
 * here we use uprobes to hook into each call of `print_time`
 * When we get into the function call we print out a message
 */
uprobe:/usr/local/bin/gotime_bpftrace:main.print_time {
 print("print_time was called")
}
```

Cool, now lets write a little helper script which will compile the binary and run our BPFTrace script!

```bash

#!/bin/bash
set -e

go build -o gotime_bpftrace example_1.go
# we need this to be in a known location, notice above how
# the uprobe attaches to the full path of the binary, feel
# free to change these
sudo cp gotime_bpftrace /usr/local/bin/
# This will run gotime_bpftrace Which is our binary from 
# example_1.go and our bpftrace program
parallel --ungroup ::: "gotime_bpftrace" "sudo ./example_1.bt"
```

If all went well you should have seen something like this:

```
./run.sh
[sudo] password for paul-montag:
Attaching 1 probe...
(time.Time) { wall: (uint64) 13959114213583033417, ext: (int64) 1000273918, loc: (*time.Location)(0x596740)({ <max depth reached> }) }
print_time was called
(time.Time) { wall: (uint64) 13959114214656847425, ext: (int64) 2000346091, loc: (*time.Location)(0x596740)({ <max depth reached> }) }
print_time was called
(time.Time) { wall: (uint64) 13959114215730869990, ext: (int64) 3000626847, loc: (*time.Location)(0x596740)({ <max depth reached> }) }
print_time was called

...

```

Neat! But, we don't want to just know that print_time was called, we want to actually see the time which was sent.

## The Time

The content starting with `(time.Time)` is what spew is outputting from the Go app itself. The data might make a bit more sense if we look at the struct definition:

```go
// Time stores the clock time in `wall` and `ext`, with the location
// stored in `loc`
type Time struct {
	wall uint64
	ext  int64
	loc *Location
}
```

So spew was just printing the internals of a Time struct. But, how do we turn a `wall` and `ext` into something useful, like a human readable time? Luckily the comments above tell us exactly what to do.

> wall and ext encode the wall time seconds, wall time nanoseconds,
> and optional monotonic clock reading in nanoseconds.
>
> From high to low bit position, wall encodes a 1-bit flag (hasMonotonic),
> a 33-bit seconds field, and a 30-bit wall time nanoseconds field.
> The nanoseconds field is in the range [0, 999999999].
> If the hasMonotonic bit is 0, then the 33-bit field must be zero
> and the full signed 64-bit wall seconds since Jan 1 year 1 is stored in ext.
> If the hasMonotonic bit is 1, then the 33-bit field holds a 33-bit
> unsigned wall seconds since Jan 1 year 1885, and ext holds a
> signed 64-bit monotonic clock reading, nanoseconds since process start.

Oh come on! That is a nasty amount of bit shifting and total nonsense to get from those two values to a human readable date of some kind. Well, lets start by getting the values from the function call.

## The Registers

Go uses a mixture of CPU registers and the stack to move arguments from one function to another. For more specific information dig into [the go ABI](https://tip.golang.org/src/cmd/compile/abi-internal#function-call-argument-and-result-passing). This is at least true for the version of go I am using. They are willing to change these things whenever a need arises.

When we pass a struct as a function parameter each piece of the struct is broken down and sent in a different register or stack location. Here we have `wall` a `uint64`, `ext` a `int64` and `loc` a pointer to some struct somewhere. Since each of these is 64bits, we should find each value in a register. Let's use objdump to figure it out.

```
bjdump gotime_bpftrace --disassemble=main.main

00000000004b69a0 <main.main>:
...
  4b6a25:	48 8b 44 24 20       	mov    0x20(%rsp),%rax
  4b6a2a:	48 8b 5c 24 28       	mov    0x28(%rsp),%rbx
  4b6a2f:	48 8b 4c 24 30       	mov    0x30(%rsp),%rcx
  4b6a34:	e8 27 00 00 00       	call   4b6a60 <main.print_time>
...
```

I went ahead and removed all the non important bits. Above you can see the assembly setting up the registers for the function call to `print_time`. Then at address `4b6a34` we make the function call.

`0x20(%rsp)` defines an offset from the stack pointer. Need a refresher on the stack? [check out this video](https://www.youtube.com/watch?v=CRTR5ljBjPM). Above you can see us referencing 3 parts of the stack and moving them into a cpu register (`%rax`, `%rbx`, `%rcx`). Our struct also had 3 values in it! That must mean:

```go
// The following comments assume we have set up or function
// parameters but haven't yet called the function print_time
type Time struct {
	// wall is stored at %rsp+32 or in hex %rsp+0x20 and is
	// in the CPU Register rax it takes up 8 bytes
	wall uint64
	// ext is stored 8 bytes later at %rsp+40 or in hex
	// %rsp+0x28. It is in CPU register rbx
	ext  int64
	// loc is stored another 8 bytes later at %rsp+48 or in
	// hex %rsp+0x30. It is in CPU register rcx
	loc *Location
}
```

But wait?! How do we know our struct fields are in the right order? Some languages disregard the order your source code defines struct fields so it can optimize the [alignment](https://en.wikipedia.org/wiki/Data_structure_alignment) and reduce memory consumption. Luckily for us it seems Go has decided to [not take that approach](https://en.wikipedia.org/wiki/Data_structure_alignment).

Armed with this info, we should be able to take the values from the registers and see the same data.

```
#!/usr/bin/bpftrace

/*
 * here we use uprobes to hook into each call of `print_time`
 * When we get into the function call we print out a message
 */
uprobe:/usr/local/bin/gotime_bpftrace:main.print_time {
 printf(
  "(bpft.Time) { wall: (uint64) %llu, ext: (int64) %lld, loc: (*time.Location)(0x%x)({ <max depth reached> })  } \n",
  reg("ax"), /* grab `wall` from the register */
  reg("bx"), /* grab `ext` from the register */
  reg("cx") /* grab `loc` from the register */
 );
}
```

We formatted it the same as the applications output so it's easy to validate.

```
 ./run.sh
Attaching 1 probe...
(time.Time) { wall: (uint64) 13959119351788076429, ext: (int64) 1000244421, loc: (*time.Location)(0x596740)({ <max depth reached> }) }
(bpft.Time) { wall: (uint64) 13959119351788076429, ext: (int64) 1000244421, loc: (*time.Location)(0x596740)({ <max depth reached> })  }

(time.Time) { wall: (uint64) 13959119352861988601, ext: (int64) 2000414772, loc: (*time.Location)(0x596740)({ <max depth reached> }) }
(bpft.Time) { wall: (uint64) 13959119352861988601, ext: (int64) 2000414772, loc: (*time.Location)(0x596740)({ <max depth reached> })  }
```

## The Time!

Oh yeah, we still need to turn this into an actual time. We could implement the algorithm in BPFTrace's scripting language, but I tried and it is so ugly I couldn't finish it. I even wrote some C which would allow me to parse a time from struct and then output the date, but, as far as I can tell at least, BPFtrace lets you include c source and header files just for struct definitions. You can't run arbitrary code with it. Sad.

So let's use the code that is really good at dealing with this implementation of time. Go!

The idea with the following code is to read JSON from the output of our BPFTrace script. Then we can make a `time.Time` from that and render it nicely with go.

```go
package main

import (
	"bufio"
	"encoding/json"
	"fmt"
	"os"
	"time"
	"unsafe"
)

func main() {
	// read in each line, we will be expecting new line
	// delimited JSON.
	s := bufio.NewScanner(os.Stdin)
	s.Split(bufio.ScanLines)

	// buf has the same structure as time.Time, but it's
	// fields are exported, and it has helpful JSON tags
	var buf struct {
		Wall     uint64 `json:"wall"`
		Ext      int64  `json:"ext"`
		location *time.Location
	}

	for s.Scan() {
		// unmarshal the data into the buff
		err := json.Unmarshal(s.Bytes(), &buf)
		if err != nil {
			// if we can't unmarshal then output
			// the original text to the screen
			fmt.Println(s.Text())
			continue
		}

		// buf has the same in memory layout of
		// time.Time, so we can do an unsafe type
		// conversion to turn it into a time.Time
		//
		// `unsafe.Pointer(&buf)` turn the pointer to
		// buf into an unsafe pointer
		//
		// `(*time.Time)(unsafe.Pointer(&buf))` do
		// an unsafe cast into a pointer to time.Time
		//
		// `*(*time.Time)(unsafe.Pointer(&buf))`
		// dereference the `*time.Time` so we are left
		// with a time.Time object
		ti := *(*time.Time)(unsafe.Pointer(&buf))
		// print out our time.
		fmt.Println(ti.String())
	}
}
```

We also have to modify our BPFTrace script to output JSON. We can drop the `loc` since we aren't going to be using it to parse the time.

```
#!/usr/bin/bpftrace

uprobe:/usr/local/bin/gotime_bpftrace:main.print_time {
 printf(
  "{ \"wall\": %llu, \"ext\": %lld }\n",
  reg("ax"), /* grab `wall` */
  reg("bx") /* grab ext */
 );
}
```

Now we can modify our run script a little:

```bash
#!/bin/bash
set -e

go build -o gotime_bpftrace example_1.go
# added step to compile parse_time
go build -o parse_time parse_time.go
sudo cp gotime_bpftrace /usr/local/bin/
# pipe output of bpftrace script into parse_time
parallel --ungroup ::: "gotime_bpftrace" "sudo ./example_1.bt | ./parse_time"
```

I am also going to remove everything in our Go application so we only see the output from BPFTrace.

```go
package main

import (
	"time"
)

func main() {
	// create a new ticker which will create a time every
	// second
	tick := time.NewTicker(time.Second)
	for {
		t := <-tick.C
		// call out to print time
		print_time(t)
	}
}

//go:noinline
func print_time(time time.Time) {
}
```

Finally we get the following output:

```
./run.sh
Attaching 1 probe...
2024-10-06 13:34:36.508891113 +0000 UTC m=+1.000260446
2024-10-06 13:34:37.508996638 +0000 UTC m=+2.000366006
2024-10-06 13:34:38.509231684 +0000 UTC m=+3.000601051
2024-10-06 13:34:39.509390572 +0000 UTC m=+4.000759933
2024-10-06 13:34:40.509559909 +0000 UTC m=+5.000929278
```

## The Conclusion

This project was far more complicated than I thought it would be. I was hoping to complete this in an afternoon, but somehow this work spanned, on and off, over a few months. Turns out C and Go have a similar, **but different**, [format specifiers](https://www.geeksforgeeks.org/format-specifiers-in-c/), but go is [far more forgiving](https://pkg.go.dev/fmt#hdr-Printing) since reflection allows go to understand the data it is working with. C is just bits, you don't have to cast shit, if you say `%d` those bits are rendered as a signed integer type. Something obvious in hindsight, but it sent me down a rabbit hole of wrong theories which took time.

I also made many assumptions about BPFtrace which now seem untrue. I had hopped that by escaping to C I could do more complicated things and render the time right out of BPFTrace, but there doesn't seem to be any "escaping to c". Any headers and C source is there for defining structs; which you [can't instantiate](https://github.com/bpftrace/bpftrace/blob/master/man/adoc/bpftrace.adoc#structs) within a BPFtrace program. Again, in hindsight this made a lot of sense.

I don't intend that to be a critique of BPFTrace, since introducing more complexity could lead to a bloated complicated product that is even scarier than it already is. BPFTrace is intended to let you peer into the state of your application while it runs, and it does that very well. They should keep things simple and language agnostic which they have done thus far.

Anyway, I think we're done here. Time to move onto something else.
