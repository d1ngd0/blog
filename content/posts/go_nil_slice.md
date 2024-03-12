+++
title = 'Go Nil Slice'
date = 2024-03-12T05:32:07-05:00
draft = false
+++

```go
var names []string

for _, person := range bus.People() {
	names = append(names, person.name)
}
```

This probably looks pretty familiar. If it doesn't and you are like "OMG, how is that working?!?! Doesn't that nil panic?!", no worries, `append` will instantiate the slice for you. However you are leaving it up to Golang to guess how many items will end up in the final slice, and it has 0 context. Let's dive into what a slice is under the hood.

## Slice Definition

This is a topic a plethora of bloggers have gone through, so I will keep this as terse as possible. If what I say doesn't make sense jump over to the [go blog](https://go.dev/blog/slices-intro) to get a better description.

```go
type slice struct {
	array unsafe.Pointer
	len int
	cap int
}
```

Above is the definition of a slice, we have a pointer `array`, in memory, to the beginning of the array; AKA the actual data that we think of when we think of a slice. Then we have the `len` (length) of the slice. Note: that is the length of the slice **not the array** this distinction is important (you will see later). Finally you have the `cap` which is the length of the underlying array. The Gunthers of the world might feel this is wasteful, "Give me the unsafe pointer, index overflow is a skill issue." Well Gunther, the [Whitehouse has something to say about that](https://www.youtube.com/watch?v=T4ZUMvALdKI). ([actual paper](https://www.whitehouse.gov/wp-content/uploads/2024/02/Final-ONCD-Technical-Report.pdf))

Anyway, when `append`ing to an existing slice go checks the `cap` to see if adding another value would write outside the bounds of the slice. If `len++ > cap` we have to allocate a new slice, copy the data over to that new slice and then add our new value. However, if there is room go can just slap that bad boy <sub>(or girl (all genders can be bad (I am digging a deeper whole aren't I?)))</sub> onto the end of the array and increase the value of `len`.

When creating a slice you have the *option* of supplying a cap, which sets the initial length of the underlying array.

```
//       make(type,     len, cap)
names := make([]string, 0,   100)
```

Now you will have a slice which can be `append`ed to 100 times before having to create a new underlying array.

## That's Neat, Who Cares

OK, yeah, onto the point. That whole copy the data into a new slice thing; Yeah it can be expensive. Often slices get tossed over to the heap adding to the performance hit. Who cares what I say, lets see what benchmarks say:

```
❯ go test . -test.run='^$' -test.bench='^Benchmark(Nil|NonEmpty)Slice'
goos: linux
goarch: amd64
pkg: github.com/d1ngd0/blog_work
cpu: AMD FX(tm)-8320 Eight-Core Processor
BenchmarkNonEmptySlice1000-8   	   31526	     38780 ns/op	   41088 B/op	       4 allocs/op
BenchmarkNonEmptySlice500-8    	   56721	     20679 ns/op	   22656 B/op	       3 allocs/op
BenchmarkNonEmptySlice250-8    	  107714	      9528 ns/op	   10368 B/op	       2 allocs/op
BenchmarkNonEmptySlice100-8    	 5320218	       235.5 ns/op	       0 B/op	       0 allocs/op
BenchmarkNonEmptySlice47-8     	 8044794	       141.8 ns/op	       0 B/op	       0 allocs/op
BenchmarkNonEmptySlice10-8     	18386499	        65.56 ns/op	       0 B/op	       0 allocs/op
BenchmarkNonEmptySlice1-8      	24260070	        51.28 ns/op	       0 B/op	       0 allocs/op
BenchmarkNilSlice1000-8        	   35143	     35947 ns/op	   35184 B/op	      11 allocs/op
BenchmarkNilSlice500-8         	   61804	     19340 ns/op	   18800 B/op	      10 allocs/op
BenchmarkNilSlice250-8         	  107950	     10615 ns/op	    9328 B/op	       9 allocs/op
BenchmarkNilSlice100-8         	  196062	      5543 ns/op	    4464 B/op	       8 allocs/op
BenchmarkNilSlice47-8          	  368827	      3275 ns/op	    2160 B/op	       7 allocs/op
BenchmarkNilSlice10-8          	 1000000	      1164 ns/op	     496 B/op	       5 allocs/op
BenchmarkNilSlice1-8           	10545158	       154.0 ns/op	      16 B/op	       1 allocs/op
PASS
ok  	github.com/d1ngd0/blog_work	22.383s
```

You can see how these [benchmarks implemented our initial example](https://github.com/d1ngd0/blog_work/blob/main/nil_slice_test.go) at my GitHub. In short we have `NonEmpty` slices created with an underlying array with a `cap` of 100. The `Nil` slices have no initial definition. You can see early on allocating a decent guess performs better than no guess. When your guess is exactly right (100) you see big performance gains `235ns < 5543ns`. However as your guess drifts further away from correct Golang's underlying algorithm starts outperforming your bad, frankly Gunther like guess of the size.

To confirm this hypothesis I adjusted the initial `cap` to `1000` and you can see the pre allocated array outperforming nil again.

```
❯ go test . -test.run='^$' -test.bench='^Benchmark(Nil|NonEmptyAdjusted)Slice'
goos: linux
goarch: amd64
pkg: github.com/d1ngd0/blog_work
cpu: AMD FX(tm)-8320 Eight-Core Processor
BenchmarkNilSlice1000-8                	   33852	     36093 ns/op	   35184 B/op	      11 allocs/op
BenchmarkNilSlice500-8                 	   59600	     19554 ns/op	   18800 B/op	      10 allocs/op
BenchmarkNilSlice250-8                 	  112942	     10621 ns/op	    9328 B/op	       9 allocs/op
BenchmarkNilSlice100-8                 	  195759	      5687 ns/op	    4464 B/op	       8 allocs/op
BenchmarkNilSlice47-8                  	  403456	      3311 ns/op	    2160 B/op	       7 allocs/op
BenchmarkNilSlice10-8                  	 1019366	      1200 ns/op	     496 B/op	       5 allocs/op
BenchmarkNilSlice1-8                   	 7731250	       155.2 ns/op	      16 B/op	       1 allocs/op
BenchmarkNonEmptyAdjustedSlice1000-8   	  447943	      2833 ns/op	       0 B/op	       0 allocs/op
BenchmarkNonEmptyAdjustedSlice500-8    	  574732	      1809 ns/op	       0 B/op	       0 allocs/op
BenchmarkNonEmptyAdjustedSlice250-8    	  943290	      1159 ns/op	       0 B/op	       0 allocs/op
BenchmarkNonEmptyAdjustedSlice100-8    	 1242712	       953.6 ns/op	       0 B/op	       0 allocs/op
BenchmarkNonEmptyAdjustedSlice47-8     	 1356798	       880.9 ns/op	       0 B/op	       0 allocs/op
BenchmarkNonEmptyAdjustedSlice10-8     	 1583610	       800.3 ns/op	       0 B/op	       0 allocs/op
BenchmarkNonEmptyAdjustedSlice1-8      	 1453552	       748.2 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	github.com/d1ngd0/blog_work	22.719s
```

## How the Slice Grows

It was quite interesting that the nil slice began outperforming the "bad" guess at a reasonable cap. I decided to look into the function that grows the underlying array to see if any hints could point out the reason.

The function that does the magic. I am going to add a bunch of comments on top of what is already there. If you want the [original](https://github.com/golang/go/blob/2ab9218c86ed625362df5060f64fcd59398a76f3/src/runtime/slice.go#L267-L299) it is on GitHub.

```go
// nextslicecap computes the next appropriate array length.
func nextslicecap(newLen, oldCap int) int {
	// start by assuming the cap is big enough, test
	// later to see if that is actually true
	newcap := oldCap
	// calculate what double the cap would be
	doublecap := newcap + newcap

	// if the new length is greater than double the cap
	// we should just return that value. This is what would
	// happen if you ran something like `append([]int{1}, []int{1,2,3})`
	// nextslicecap would be called with (newLen = 4, oldCap = 1) so this
	// function would return 4.
	if newLen > doublecap {
		return newLen
	}

	// if our oldCap is less than 256 we just want to return double. This means
	// in our previous example (starting at 100), we would return 100, 200, 400
	// and then fall into the for loop instead of returning double. We would
	// calculate a cap of 400 because we are checking the oldCap in the if
	// statement not the doublecap.
	const threshold = 256
	if oldCap < threshold {
		// just keep doub
		return doublecap
	}

	// now we switch t a 1.25x growth for anything with an oldCap greater than
	// 256.
	for {
		// Transition from growing 2x for small slices
		// to growing 1.25x for large slices. This formula
		// gives a smooth-ish transition between the two.
		newcap += (newcap + 3*threshold) >> 2

		// We need to check `newcap >= newLen` and whether `newcap` overflowed.
		// newLen is guaranteed to be larger than zero, hence
		// when newcap overflows then `uint(newcap) > uint(newLen)`.
		// This allows to check for both with the same comparison.
		if uint(newcap) >= uint(newLen) {
			break
		}
		// Paul Here: For some more context when they say overflow, they
		// mean integer overflow. Say you have an array that is nearly larger than
		// an int32 on a 32 bit machine. When you multiply that by 1.25 it could
		// overflow.
		// because newcap and old cap are ints, when you cast them into uints
		// they will maintain the same value if they are positive. However if
		// they are negative they will get a bigger number. One that isn't a valid
		// int, but the condition will work.
		// https://go.dev/play/p/c6xjvE-sFlb
	}

	// Set newcap to the requested cap when
	// the newcap calculation overflowed.
	if newcap <= 0 {
		return newLen
	}

	return newcap
}
```

Knowing all this, the subtle differences in our maths and doubling likely lead to a difference in performance over time. I'm sure someone could write their thesis on what is going on here, but I am trying to get this blog done in a single morning. :)

### Conclusion

Does any of this matter? Not really. I usually prefer to create nil slices since they look prettier. Performance isn't a problem until it's a problem. However, if you have a good guess on size within a hot path, it is a decent step to take to **potentially** speed things up. If you take anything away from this blog post it should be even when you understand how something works, subtleties can cause big swings in performance. So benchmark your particular thing and follow the numbers.
