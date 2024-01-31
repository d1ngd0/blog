+++
title = 'Go Heap Allocations'
date = 2024-01-24T21:45:13-06:00
draft = false
+++

Recently I found myself caring about performance. Usually, with go, you don't have to care if something is on the stack or the heap. Since Go is a garbage collected language all those details are taken care of for you. Worrying about these things is a pre-optimization, and the community is very happy to tell you so. However, there is the rare event performance takes precedence, in which case the favor of garbage collection turns into a frustrating scavenger hunt.

Lets write a simple benchmark so we can start looking for allocations.

<sub>main_test.go</sub>
```go
type Person struct {
	name   string
	age    int
}

func BenchmarkNew(b *testing.B) {
	for i := 0; i < b.N; i++ {
		e := Person{
			name: "eric",
			age:  12,
		}

		_, err := json.Marshal(e)
		if err != nil {
			b.FailNow()
		}
	}
}
```

We want to run our benchmarks and capture memory statistics. To do this we can use the `-memprofile` flag for `go test`. This will generate a memory profile which will help find where our allocations are occurring. We can also get a count of allocations to the heap during each loop in our benchmark by passing the `-benchmem` flag.

```
❯ go test -run='^$' -bench=BenchmarkNew -memprofile=mem.out -benchmem .
goos: linux
goarch: amd64
pkg: github.com/d1ngd0/go-play
cpu: AMD FX(tm)-8320 Eight-Core Processor
BenchmarkNew-8   	 3484684	       443.3 ns/op	      32 B/op	       2 allocs/op
PASS
ok  	github.com/d1ngd0/go-play	1.906s
```

So our code has 2 memory allocations per run. We can use the `go tool pprof` command to look through our memory profile and figure out where exactly the allocations are occurring.

```
❯ go tool pprof mem.out
File: go-play.test
Type: alloc_space
Time: Jan 24, 2024 at 10:17pm (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof)
```

Running the command above will place you in a prompt with commands to help you explore the data. The best place to start is with `top`, which shows you the top bytes allocated.

```
(pprof) top
Showing nodes accounting for 138.50MB, 100% of 138.50MB total
      flat  flat%   sum%        cum   cum%
  101.50MB 73.29% 73.29%   138.50MB   100%  github.com/d1ngd0/go-play.BenchmarkNew
      37MB 26.71%   100%       37MB 26.71%  encoding/json.Marshal
         0     0%   100%   138.50MB   100%  testing.(*B).launch
         0     0%   100%   138.50MB   100%  testing.(*B).runN
```

We can then view the **exact** location by using the `list` command.

```
(pprof) list encoding/json.Marshal
Total: 138.50MB
ROUTINE ======================== encoding/json.Marshal in /usr/local/go/src/encoding/json/encode.go
      37MB       37MB (flat, cum) 26.71% of Total
         .          .    158:func Marshal(v any) ([]byte, error) {
         .          .    159:	e := newEncodeState()
         .          .    160:	defer encodeStatePool.Put(e)
         .          .    161:
         .          .    162:	err := e.marshal(v, encOpts{escapeHTML: true})
         .          .    163:	if err != nil {
         .          .    164:		return nil, err
         .          .    165:	}
      37MB       37MB    166:	buf := append([]byte(nil), e.Bytes()...)
         .          .    167:
         .          .    168:	return buf, nil
         .          .    169:}
         .          .    170:
         .          .    171:// MarshalIndent is like Marshal but applies Indent to format the output.
```

Now we see that `buf := append([]byte(nil), e.Bytes()...)` is the line causing the heap allocation. This makes sense, as the code here is effectively copying the bytes from one slice into a new one.

One of the issues here is we are tracking heap allocations by bytes allocated,not total number of allocations. Each allocation has overhead, as our program has to find a space large enough to hold our value. Disproportionate bytes needed for each allocation to the heap may throw off our numbers, and make a small performance issue look like a huge one.

For instance lets add a big heap allocation to our benchmark.

<sub>main_test.go</sub>
```go
func BenchmarkNew(b *testing.B) {
	for i := 0; i < b.N; i++ {
		// make a big heap allocation
		v := make([]byte, 100000)
		// make sure we use v so it doesn't optimise out
		_ = v

		e := Person{
			name: "eric",
			age:  12,
		}

		// cause more heap allocations
		for x := 0; x < 1000; x++ {
			_, err := json.Marshal(e)
			if err != nil {
				b.FailNow()
			}
		}
	}
}
```

We now see a staggering number of allocations, since we have increased the number of times we run json.Marshal

```
❯ go test -run='^$' -bench=BenchmarkNew -memprofile=mem.out -benchmem .
goos: linux
goarch: amd64
pkg: github.com/d1ngd0/go-play
cpu: AMD FX(tm)-8320 Eight-Core Processor
BenchmarkNew-8   	     163	   8571304 ns/op	  138551 B/op	    2001 allocs/op
PASS
ok  	github.com/d1ngd0/go-play	2.138s
```

After looking at the allocations in `BenchmarkNew` we see this

```
(pprof) list BenchmarkNew
Total: 32.34MB
ROUTINE ======================== github.com/d1ngd0/go-play.BenchmarkNew in /home/paul/Projects/go-play/main_test.go
   30.37MB    32.24MB (flat, cum) 99.69% of Total
         .          .     13:func BenchmarkNew(b *testing.B) {
         .          .     14:	for i := 0; i < b.N; i++ {
         .          .     15:		// make a big heap allocation
   24.78MB    24.78MB     16:		v := make([]byte, 100000)
         .          .     17:		// make sure we use v so it doesn't optimise out
         .          .     18:		_ = v
         .          .     19:
         .          .     20:		e := Person{
         .          .     21:			name: "eric",
         .          .     22:			age:  12,
         .          .     23:		}
         .          .     24:
         .          .     25:		// cause more heap allocations
         .          .     26:		for x := 0; x < 1000; x++ {
    5.58MB     7.46MB     27:			_, err := json.Marshal(e)
         .          .     28:			if err != nil {
         .          .     29:				b.FailNow()
         .          .     30:			}
         .          .     31:		}
         .          .     32:	}
```

You can see line 16 has a much larger allocation in bytes, but each run should cause a single allocation. We are running 1000 allocations in the for loop in BenchmemNew and another 1000 inside `json.Marshal`. This will have a much larger performance impact. Though the way we are measuring things would make us look at line 16 first. Let's run our test again to get a count of allocations instead.

First when we run our command we will set the flag `-memprofilerate=1`. This will count **every** allocation that occurs, though it does this at a massive performance cost. Since we only care to count allocations, we don't really care how long it takes to run.

```
❯ go test -run='^$' -bench=BenchmarkNew -memprofile=mem.out -memprofilerate=1 -benchmem .
```

Now we can run `pprof` again, but this time we will pass in the `-alloc_objects` flag.

```
❯ go tool pprof -alloc_objects mem.out
(pprof) list BenchmarkNew
Total: 396519
ROUTINE ======================== github.com/d1ngd0/go-play.BenchmarkNew in /home/paul/Projects/go-play/main_test.go
    264264     396342 (flat, cum)   100% of Total
         .          .     13:func BenchmarkNew(b *testing.B) {
         .          .     14:	for i := 0; i < b.N; i++ {
         .          .     15:		// make a big heap allocation
       264        264     16:		v := make([]byte, 100000)
         .          .     17:		// make sure we use v so it doesn't optimise out
         .          .     18:		_ = v
         .          .     19:
         .          .     20:		e := Person{
         .          .     21:			name: "eric",
         .          .     22:			age:  12,
         .          .     23:		}
         .          .     24:
         .          .     25:		// cause more heap allocations
         .          .     26:		for x := 0; x < 1000; x++ {
    264000     396078     27:			_, err := json.Marshal(e)
         .          .     28:			if err != nil {
         .          .     29:				b.FailNow()
         .          .     30:			}
         .          .     31:		}
         .          .     32:	}
```

Now we can see where the allocations are really occurring.
