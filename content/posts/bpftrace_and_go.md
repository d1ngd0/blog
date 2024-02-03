+++
title = 'BPFTrace and Go'
date = 2024-01-28T20:45:05-06:00
draft = false
+++

You know what is awesome! BPFtrace! There is nothing more magic than showing someone exactly what is happening in their application, **RIGHT AT THAT MOMENT**. There is something neat about peering under the hood of a running application and finding that your pristine software, your baby which you have labored over making perfect is still marred with logical errors that don't show themselves when viewed holistically. Sadly, when you are reaching for something like BPFtrace you are typically in a bind, and the complexity of such a tool is frustrating. This is compounded by the fact that Go, in no way, makes your life easier here. Hence I will write down what I know in hopes to clear someone else's frustrations. (more likely my future self)

Tools exist to try and help: Steven Johnstone wrote up the [go-bpf-gen](https://github.com/stevenjohnstone/go-bpf-gen) tool to help generate BPF scripts from templates. This acted, for me at least, as a Rosetta stone on my path to learning many of the things here, since golang has some [peculiarities](https://github.com/stevenjohnstone/go-bpf-gen#why) that make using BPFtrace just a little harder. While this tool worked I didn't want to rely on it, mostly out of a stubborn need to understand.

Let's get into it already: I will be using go version `go1.21.5 linux/amd64`. **This matters**: Go made a change in v1.17 from [stack based to CPU register based function parameters](https://go.dev/doc/go1.17#compiler). So if you are trying to run BPFtrace on an old binary you will have to pull stuff out of the stack.

<sub>psst. How do you figure out what version it was compiled with: [I got ya don't worry](https://stackoverflow.com/questions/18990242/find-out-the-version-of-go-a-binary-was-built-with)</sub>

Also: I am going to assume you know Go already. \<sarcasm\>It is a *bold assumption* given you are digging into "BPFtrace" and "Go".\</sarcasm\> If you hope I explain every detail of the go code you will be disappointed, though I will keep my examples concise.

## Kinda Like Print Debugging

You know when you are trying to figure out if some function is even being called, so you recompile with a nice little message letting you know it ran:

```go
func main() {
	for {
		neatFunction()
		time.Sleep(1 * time.Second)
	}
}

//go:noinline
func neatFunction() {
	fmt.Println("OMG RUN!!!!!")
}
```

You can use BPFtrace to help out. Instead of recompiling your program you can write a quick BPFtrace script which makes use of uprobes. This enables you to hook into userland programs and execute extra logic at runtime. Here we run additional logic to write "here" whenever the `neatFunction` is called.

```
sudo bpftrace -e 'uprobe:/home/paul/Projects/go-play/go-play:main.neatFunction { 
  print("here"); 
}'
```

This is extra handy when you don't have a development environment set up, but are debugging on a running system. What about when you print out `Here 1`, `Here 2`, `Here 3` within a function. You know, those times experienced annoying engineers say "Well, have you, maybe, thought of using, I don't know, a debugger!?!?!?!" I don't like your snooty tone GUNTHER!!! Maybe I can just print my way out instead of learning [delve](https://github.com/go-delve/delve) GuNtHeR!!! MAYBE I CAN'T STOP THE WORLD WHENEVER THERE IS AN ISSUE GUNTHER!!!! Sorry, lets just get to the example:

```go
func main() {
	for {
		// there is a bug here where it will always output false
		// since it can only output a single value, 0. So you will see
		// no randomness. However I am like 95% done at this point so
		// the bug stays put
    test := rand.Intn(1) == 1
		neatFunction(test, test, !test)
		time.Sleep(1 * time.Second)
	}
}

//go:noinline
func neatFunction(cond1, cond2, cond3 bool) {
	if cond1 {
		fmt.Println("here 1")
	}

	fmt.Println("here 1.5")
	// probably some other bad code

	fmt.Println("here PLEASE!!!")

	if cond2 {
		fmt.Println("here 2")
	}

	// clever code that will trip up
	// another engineer who will curse
	// your family

	if cond3 {
		fmt.Println("here 3")
	}

	fmt.Println("here 4")
}
```

"Real Programmers" love to shit on this... Let me tell you, reaching for a debugger is sometimes the right thing to do, and even though GuNThEr is right that you should learn [delve](https://github.com/go-delve/delve), this is a perfectly reasonable solution. However, if you are debugging in a test or prod environment, you can't keep recompiling and launching new binaries. So how can we re-create the logic above with BPFtrace? By reading the assembly! (Oh God I am sorry)

So to start lets grab the symbol we are looking for:

```
❯ objdump -t go-play | grep neatFunction
000000000047b0a0 g     F .text	00000000000001e9 main.neatFunction
```

Now we can dump the assembly:

```
❯ objdump --disassemble=main.neatFunction go-play

go-play:     file format elf64-x86-64


Disassembly of section .text:

000000000047b0a0 <main.neatFunction>:
  47b0a0:	4c 8d 64 24 f0       	lea    -0x10(%rsp),%r12
  47b0a5:	4d 3b 66 10          	cmp    0x10(%r14),%r12
  [A bunch more assembly]
```

Now, I am no assembly expert, and you don't have to be either because ChatGPT exists. We just need to know what address we want to hijack. Lets find all the places our if statements are running by searching for `test`. I am also going to include the next instruction after test, which will jump to a new location in the program based on the previous condition:

```go
❯ objdump --disassemble=main.neatFunction go-play | grep test -A 1
  482728:	84 c0                	test   %al,%al
  48272a:	74 41                	je     48276d <main.neatFunction+0x6d>
--
  4827f1:	84 d2                	test   %dl,%dl
  4827f3:	74 3e                	je     482833 <main.neatFunction+0x133>
--
  48283b:	84 d2                	test   %dl,%dl
  48283d:	74 3e                	je     48287d <main.neatFunction+0x17d>
```

Look at that, three matches, just like our three conditions! And we even get symbols to where it jumps to. **BUT HOLD ON**: With assembly things are often not what they seem. In the first example we compare `%al` to itself. This sets the appropriate flags for our next jump instruction. If the Zero Flag (ZF) is set, meaning that the result of the previous TEST operation indicated that `%al` is zero, the program will jump to the address `48276d`. In other words, we jump when false! This is used to skip instructions... the instructions skipped are the contents of our if statement. This becomes a whole lot clearer when using `--visualize-jumps=color`

```
❯ objdump --disassemble=main.neatFunction --visualize-jumps=color go-play
--	
  48283b:	|  |      84 d2                	test   %dl,%dl
  48283d:	|  |  /-- 74 3e                	je     48287d <main.neatFunction+0x17d>
  48283f:	|  |  |   44 0f 11 7c 24 38    	movups %xmm15,0x38(%rsp)
  482845:	|  |  |   48 8d 15 d4 82 00 00 	lea    0x82d4(%rip),%rdx        # 48ab20 <type:*+0x7b20>
  48284c:	|  |  |   48 89 54 24 38       	mov    %rdx,0x38(%rsp)
  482851:	|  |  |   4c 8d 05 48 d1 03 00 	lea    0x3d148(%rip),%r8        # 4bf9a0 <runtime.buildVersion.str+0x50>
  482858:	|  |  |   4c 89 44 24 40       	mov    %r8,0x40(%rsp)
  48285d:	|  |  |   48 8b 1d cc 05 0b 00 	mov    0xb05cc(%rip),%rbx        # 532e30 <os.Stdout>
  482864:	|  |  |   48 8d 05 2d d6 03 00 	lea    0x3d62d(%rip),%rax        # 4bfe98 <go:itab.*os.File,io.Writer>
  48286b:	|  |  |   48 8d 4c 24 38       	lea    0x38(%rsp),%rcx
  482870:	|  |  |   bf 01 00 00 00       	mov    $0x1,%edi
  482875:	|  |  |   48 89 fe             	mov    %rdi,%rsi
  482878:	|  |  |   e8 03 71 ff ff       	call   479980 <fmt.Fprintln>
  48287d:	|  |  \-> 44 0f 11 7c 24 28    	movups %xmm15,0x28(%rsp)
  482883:	|  |      48 8d 15 96 82 00 00 	lea    0x8296(%rip),%rdx        # 48ab20 <type:*+0x7b20>
--
```

Will all of your go if statements be turned into `test` instructions? Probably not, but I wasn't kidding about ChatGPT. Paste the whole function in there and it will explain it line by line. My guess is you will find a spot that interests you in your debugging quest.

Anyway, lets get back to BPFtrace. You can create scripts and add the path to bpftrace in the shebang (`#!`). So lets do that to write out each time we are in one of these if statements:

Now instead of grabbing the reference offset, we will just grab the next instruction after the `je`. You can do this by taking the address of the instruction you want to print as the target. This is obviously as brittle as my sense of self worth, since these locations change on any recompile.

```
#!/usr/bin/bpftrace --unsafe

uprobe:/home/paul/Projects/go-play/go-play:0x48272c {
  print("here 1 BPF");
}

uprobe:/home/paul/Projects/go-play/go-play:0x4827f5 {
  print("here 2 BPF");
}

uprobe:/home/paul/Projects/go-play/go-play:0x48283f {
  print("here 3 BPF");
}
```

```
❯ sudo ./print-here
Attaching 3 probes...
here 3 BPF
```

Nice, we can see the run only had condition 3 fire! TAKE THAT GUNTHER!!! WHO'S A "ReAl ProGrAMeR" NOW!!!! Sorry, sorry... Let's move on.

### Dumping Function Parameters

Lets start with a new go program. People like to write add functions at times like these... So let's write a minus!

```go
func main() {
	for {
		minus(22, 12)
		time.Sleep(1 * time.Second)
	}
}

//go:noinline
func minus(a, b int) int {
	return a - b
}
```

Lets dump the assembly for the main function so we can see where the parameters are being set before calling the function.

```
❯ objdump --disassemble=main.main go-play
0000000000458d60 <main.main>:
	...
  458d6e:	b8 16 00 00 00       	mov    $0x16,%eax
  458d73:	bb 0c 00 00 00       	mov    $0xc,%ebx
  458d78:	e8 23 00 00 00       	call   458da0 <main.minus>
	...
```

I have removed all the stuff that isn't part of the function call.

`mov $0x16,%eax`: The keen eyed of you may realize that `0x16` is hexadecimal for 22. The `mov` instruction is used to copy data from one place to another. The place it is copying it `%eax` is a [register](https://en.wikipedia.org/wiki/Processor_register) on the CPU. This means our `minus` function assumes our first parameter will be in register `%eax`.

`mov $0xc,%ebx`: does the same thing, but for the second argument. `0xc` is hexadecimal for 12, and we are loading 12 into `%ebx`.

`call 458da0`: calls the function. The definition for which is at address `0x458da0`.

Armed with this new knowledge we can grab the values being passed into the function like so.

```
#!/usr/bin/bpftrace

uprobe:/home/paul/Projects/go-play/go-play:main.minus {
	printf("minus(%d, %d)\n", reg("ax"), reg("bx"));
}
```

Now we can run it

```
❯ sudo ./show_value.bt
Attaching 1 probe...
minus(22, 12)
minus(22, 12)
minus(22, 12)
```

### string parameters

Sure, this is where things get a little harder. BPFtrace has a whole load of helper [functions](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#functions) one of them being the [str](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#5-str-strings) function. This takes a pointer to a string, and then the length of the string. Lets write a function that takes a string as an argument and then look at the call signature to figure out what the memory shape of a string is.

<sub>main.go</sub>
```go

func main() {
	for {
		print_me("Hello, world")
		time.Sleep(1 * time.Second)
	}
}

//go:noinline
func print_me(msg string) {
	fmt.Println(msg)
}
```

<sub>objdump</sub>
```
❯ objdump --disassemble=main.main go-play | grep 'main\.print_me' -B 2
  47b06e:	48 8d 05 d6 9f 01 00 	lea    0x19fd6(%rip),%rax        # 49504b <go:string.*+0xdd3>
  47b075:	bb 0c 00 00 00       	mov    $0xc,%ebx
  47b07a:	e8 21 00 00 00       	call   47b0a0 <main.print_me>
```

Alright, there is a lot here:

`lea 0x19fd6(%rip),%rax`: uses `lea` which is used to calculate an address. Here we are calculating the memory address of `%rip` + `0x19fd6`. That calculated value will go into CPU register `%rax`. So what the hell is `%rip`? `%rip` is a CPU register, the "instruction pointer" register to be exact. It holds the value of the next instruction to run, and is incremented automatically. So the compiler has defined the relative address of the first argument as an offset of `%rip`. If you haven't put it together yet, we are referencing the location of "Hello, world" in the binary of the application. There are some [gems in here](https://cs.brown.edu/courses/csci1310/2020/notes/l08.html) if you are further interested. Honestly, this is a bit of a tangent. All that matters is `%rax` has the pointer to the first byte of a string

`mov $0xc,%ebx`: that is better, it moves the number `12` into the `bx` register.

But why does it look like we are passing in two arguments to a function that takes a single parameter? There is a great blog post on strings [here](https://go101.org/article/string.html) but the short of it, is a strings structure looks more like this in go.

```go
type string struct {
	first *byte
	len int
}
```

So what we are seeing is the address of the first byte, and the length of the string. Well, gee golly, it looks like we have everything we need to fill our `str` function.

```
#!/usr/bin/bpftrace

uprobe:/home/paul/Projects/go-play/go-play:main.print_me {
	printf("%s\n", str(reg("ax"), reg("bx")));
}
```

<sub>When running it</sub>
```
❯ sudo ./print_string.bt
[sudo] password for paul:
Attaching 1 probe...
Hello, world
Hello, world
Hello, world
```

## Wrap it up

Yeah this has gone on long enough. BPFtrace is freaking awesome. There is nothing quite like feeling like a wizard, capable of hunting down any bug, regardless of where it appears. Plus you can show up Gunther, because that idiot is still trying to replicate the issue on his local machine with his precious f\*\*\*king debugger. Seriously though, you should learn delve.
