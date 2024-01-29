+++
title = 'Bpftrace_and_go'
date = 2024-01-28T20:45:05-06:00
draft = true
+++

You know what is awesome! BPFtrace! There is nothing more magic than showing someone exactly what is happening in their application, **RIGHT AT THAT MOMENT**. There is something neat about peering under the hood of a running application and finding that your pristine software, your baby which you have labored over making perfect is still marred with logical errors that don't show themselves when viewed hositically. Sadly, when you are reaching for something like BPFtrace you are typically in a bind, and the complexity of such a tool is frustrating. This is compounded by the fact that Go, in no way, makes your life easier here. Hence I will write down what I know in hopes to clear someone else's frustrations. (more likely my future self)

Tools exist to try and help: Steven Johnstone wrote up the [go-bpf-gen](https://github.com/stevenjohnstone/go-bpf-gen) tool to help generate BPF scripts from templates. This acted, for me at least, as a Rosetta stone on my path to learning many of the things here, since golang has some [peculiarities](https://github.com/stevenjohnstone/go-bpf-gen#why) that make using BPFtrace just a little harder. While this tool worked I didn't want to rely on it, mostly out of a stuborn need to understand.

Let's get into it already: I will be using go version `go1.21.5 linux/amd64`. **This matters**: Go made a change in v1.17 from [stack based to CPU register based function parameters](https://go.dev/doc/go1.17#compiler). So if you are trying to run BPFtrace on an old binary you will have to pull stuff out of the stack.

<sub>psst. How do you figure out what version it was compiled with: [I got ya don't worry](https://stackoverflow.com/questions/18990242/find-out-the-version-of-go-a-binary-was-built-with)</sub>

Also: I am going to assume you know Go already. \<sarcasm\>It is a *bold assumption* given you are digging into "BPFtrace" and "Go".\</sarcasm\> If you hope I explain every detail of the go code you will be disapointed, though I will keep my examples concise.

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

"Real Programmers" love to shit on this... let me tell you, reaching for a debugger is sometimes the right thing to do, and even though GuNThEr is right that you should learn [delve](https://github.com/go-delve/delve), this is a perfectly reasonable solution. However, if you are debugging in a test or prod environment, you can't keep recompiling and launching new binaries. So how can we re-create the logic above with BPFtrace? By dumping the assembly and doing offsets! (Oh God I am sorry)

So to start lets grab the symbol we are looking for:

```
❯ objdump -t go-play | grep neatFunction
000000000047b0a0 g     F .text	00000000000001e9 main.neatFunction
```

now we can dump the assembly:

```
❯ objdump --disassemble=main.neatFunction go-play

go-play:     file format elf64-x86-64


Disassembly of section .text:

000000000047b0a0 <main.neatFunction>:
  47b0a0:	4c 8d 64 24 f0       	lea    -0x10(%rsp),%r12
  47b0a5:	4d 3b 66 10          	cmp    0x10(%r14),%r12
  [A bunch more assembly]
```

Now, I am no assembly expert, and you don't have to be either because ChatGPT exists. We just need to know what our offset is going to be. Lets print out all the functions we `call` in our function:

```
❯ objdump --disassemble=main.neatFunction go-play | grep call
  47b108:	e8 53 ae ff ff       	call   475f60 <fmt.Fprintln>
  47b146:	e8 15 ae ff ff       	call   475f60 <fmt.Fprintln>
  47b184:	e8 d7 ad ff ff       	call   475f60 <fmt.Fprintln>
  47b1ce:	e8 8d ad ff ff       	call   475f60 <fmt.Fprintln>
  47b218:	e8 43 ad ff ff       	call   475f60 <fmt.Fprintln>
  47b256:	e8 05 ad ff ff       	call   475f60 <fmt.Fprintln>
  47b270:	e8 ab fa fd ff       	call   45ad20 <runtime.morestack_noctxt.abi0>
```

<sub>The call to `<runtime.morestack_noctxt.abi0>` is golang checking the stack size and increasing it if it is too small. Cool stuff, but out of scope for what I am talking about now.</sub>

Here you can see all of our original `here` debugging we are doing. We can also find all the places our if statements are running by searching for `test`. I am also going to include the next instruction after test, which will jump to a new location in the program if the condition was true:

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

Look at that, three matches, just like our three conditions! And we even get symbols to where it jumps to. We can now assume that:

```go
func neatFunction(cond1, cond2, cond3 bool) {
  // <main.neatFunction+0x6d>
	if cond1 {
		fmt.Println("here 1")
	}

  ...
  
  // <main.neatFunction+0x133>
	if cond2 {
		fmt.Println("here 2")
	}

  ...

  // <main.neatFunction+0x17d>
	if cond3 {
		fmt.Println("here 3")
	}

}
```

Will all of your go if statements be turned into `test` instructions? Probably not, but I wasn't kidding about ChatGPT. Paste the the whole function in there and it will explain it line by line. My guess is you will find a spot that interests you in your debugging quest.

Anyway, lets get back to BPFtrace. You can create scripts and add the path to bpftrace in the shebang (`#!`). So lets do that to write out each time we are in one of these if statements:

#TODO: finish this, It doesn't work yet
```
#!/usr/bin/bpftrace --unsafe

uprobe:/home/paul/Projects/go-play/go-play:main.neatFunction+0x6d {
  print("here 1 BPF");
}

uprobe:/home/paul/Projects/go-play/go-play:main.neatFunction+0x133 {
  print("here 2 BPF");
}

uprobe:/home/paul/Projects/go-play/go-play:main.neatFunction+0x17d {
  print("here 3 BPF");
}
```

TAKE THAT GUNTHER!!! WHO'S A "ReAl ProGrAMeR" NOW!!!! Sorry, sorry... Let's move on.

### Dumping Function Parameters
