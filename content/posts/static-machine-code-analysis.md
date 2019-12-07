---
title: "Static Machine Code Analysis"
date: 2019-09-03T23:54:52+02:00
---

Modern processors are complex beasts.
They reorder instructions in an ever-increasing instruction window and speculatively execute following iterations of a loop by predicting the branch of the loop condition.
Both features are meant to extract as much instruction parallelism from the program code as possible to feed superscalar CPUs with enough work.
They can execute multiple instructions in a single cycle if there are no dependencies.
Static machine code analyzers let us take a look at how our code is executed by modeling the various execution stages of a CPU.


# Pipelining

Before we stare in wonder at the output of analysis tools, let us briefly recap some basic concepts in processor design, starting with pipelining.
Let's assume we have a sequence of three instructions, each taking three cycles to be processed.
The following timeline diagram illustrates how the sequence is processed.

<svg version="1.1" width="520" height="125" xmlns="http://www.w3.org/2000/svg">
<path d="M1 30 h150 v60 h300 v30 h-150 v-60 h-300 Z" stroke="black" fill="none" stroke-width="2"/>
<g text-anchor="middle">
<text x="5"   y="20">0</text>
<text x="50"  y="20">1</text>
<text x="100" y="20">2</text>
<text x="150" y="20">3</text>
<text x="200" y="20">4</text>
<text x="250" y="20">5</text>
<text x="300" y="20">6</text>
<text x="350" y="20">7</text>
<text x="400" y="20">8</text>
<text x="450" y="20">9</text>
<text x="75"  y="50" >instruction<tspan dy="3" font-size="smaller">1</tspan></text>
<text x="225" y="80" >instruction<tspan dy="3" font-size="smaller">2</tspan></text>
<text x="375" y="110">instruction<tspan dy="3" font-size="smaller">3</tspan></text>
</g>
</svg>

The processor needs nine cycles to finish the sequence.
Can we do better?

The main insight pipelining is based upon is that one instruction is not a single block of work.
It consists of multiple smaller steps, e.g., fetching operands, executing the operation on the operands and retiring the instruction by writing the result to the destination.
Such a simple pipeline with the three stages _fetch_, _exec_ and _retire_ is illustrated in the following timeline diagram.

<svg version="1.1" width="320" height="125" xmlns="http://www.w3.org/2000/svg">
<path d="M1 30 h180 v30 h-180 Z m60 0 v60 h180 v-30 h-60 m-60 -30 v90 h180 v-30 h-60 m-60 -30 v60 m60 0 v-30" stroke="black" fill="none" stroke-width="2"/>
<g text-anchor="middle">
<text x="5" y="20">0</text>
<text x="60" y="20">1</text>
<text x="120" y="20">2</text>
<text x="180" y="20">3</text>
<text x="240" y="20">4</text>
<text x="300" y="20">5</text>
<text x="30"  y="50"  fill="blue">fetch<tspan dy="3" font-size="smaller">1</tspan></text>
<text x="90"  y="80"  fill="blue">fetch<tspan dy="3" font-size="smaller">2</tspan></text>
<text x="150" y="110" fill="blue">fetch<tspan dy="3" font-size="smaller">3</tspan></text>
<text x="90"  y="50"  fill="red">exec<tspan dy="3" font-size="smaller">1</tspan></text>
<text x="150" y="80"  fill="red">exec<tspan dy="3" font-size="smaller">2</tspan></text>
<text x="210" y="110" fill="red">exec<tspan dy="3" font-size="smaller">3</tspan></text>
<text x="150" y="50"  fill="green">retire<tspan dy="3" font-size="smaller">1</tspan></text>
<text x="210" y="80"  fill="green">retire<tspan dy="3" font-size="smaller">2</tspan></text>
<text x="270" y="110" fill="green">retire<tspan dy="3" font-size="smaller">3</tspan></text>
</g>
</svg>

After the first cycle, the _fetch_ stage of the first instruction <span style="color:blue;">fetch<sub>1</sub></span> is done.
In the next cycle, the _exec_ stage <span style="color:red;">exec<sub>1</sub></span> and, additionally, the _fetch_ stage of the next instruction <span style="color:blue;">fetch<sub>2</sub></span> can start processing.
Hence, the CPU executes parts of multiple instructions in parallel leading to a reduced execution time of the whole sequence.
Now, the processor only needs five cycles instead of nine to process the sequence of instructions.

One instruction still requires three cycles to be processed.
This is called the _latency_ of an instruction.
After the first instruction, every following instruction only needs one additional cycle to finish as the _fetch_ and _exec_ stages are already done in parallel to the previous instruction.
This is called the _throughput_ of an instruction.
It is usually stated in cycles-per-instruction (CPI).


# Execution Ports

Modern CPUs are superscalar, i.e., they can process multiple instructions in a single cycle, e.g., the `add` instruction has a throughput of 0.25 on recent Intel CPUs which means that up to four additions can be processed in a single cycle.
This is achieved by having multiple execution ports to process pipeline stages.
Some ports are specialized for certain kind of operations (e.g., load/store, division).
Other ports can perform a wide range of operations.

The number of execution ports is sometimes also called the width of a processor.
The wider it is, the more instructions it can process per cycle, theoretically.
The main issue with such monster is that it needs to be fed with enough instructions which do not have data dependencies between them.
Unsurprisingly, most programs tend to apply operations on top of each other which limits the number of independent instructions.

Out-of-order execution tries to extract as many independent sequences of instructions as possible in an instruction window which increases with every processor generation.
The independent instructions are processed in parallel (instruction level parallelism, ILP).
The results are stored to memory and made visible in the same order the instructions are laid out in the program code.
So, the observable effects look like it's processed in a sequential fashion, but under the hood, it's done in parallel and therefore faster.

{{< todo >}}
- explain execution ports, needs figure...

better: trying to understand how the CPU executes instructions, not performance prediction

Predicting the performance of a program under a given workload is very hard.
System profilers let us tap into the CPU by using performance counters.
{{< / todo >}}


{{< todo >}}
# Microcode

The last thing I have to mention before we look at the tools is microcode.

We bake a lot of operations and variations of them into a single large instruction instead of having a sequence of multiple fundamental instructions.
That is CISC vs. RISC in a nutshell.

- RISC vs. CISC
- microcode is RISC-like
- good for pipelining
{{< / todo >}}


# Intel Architecture Code Analyzer

The first tool is called [IACA][iaca].
Let us create a tiny test program and see how it works.

{{< highlight "C" >}}
#include <iacaMarks.h>

long sum(long *array, long size){
	long result = 0;
	for(long i=0; i<size; ++i){
IACA_START
		result += array[i];
	}
IACA_END
	return result;
}
{{< / highlight >}}

As IACA is a static analyzer, we do not need a full program.
A simple function is enough.
In this case, the function sums up all elements of the array passed as a pointer and size pair.
The code region we want to analyze must be marked with special instructions.
Luckily, IACA provides a header file with easy to use macros.
To analyze the throughput of the loop in _sum_, we place `IACA_START` at the beginning of the loop body and `IACA_END` right after the loop.
This way we get all instruction of the loop body and the loop condition which gets moved at the bottom of the loop by the compiler.

```
$ gcc -c -O2 sum.c -o sum.o
```

We compile the source file to an object file as usual. The object file is used as input for IACA.

```
$ iaca sum.o
Intel(R) Architecture Code Analyzer Version -  v3.0-28-g1ba2cbb build date: 2017-10-23;16:42:45
Analyzed File -  sum_simple.o
Binary Format - 64Bit
Architecture  -  SKL
Analysis Type - Throughput

Throughput Analysis Report
--------------------------
Block Throughput: 1.00 Cycles       Throughput Bottleneck: Dependency chains
Loop Count:  27
Port Binding In Cycles Per Iteration:
--------------------------------------------------------------------------------------------------
|  Port  |   0   -  DV   |   1   |   2   -  D    |   3   -  D    |   4   |   5   |   6   |   7   |
--------------------------------------------------------------------------------------------------
| Cycles |  0.5     0.0  |  0.5  |  0.5     0.5  |  0.5     0.5  |  0.0  |  0.5  |  0.5  |  0.0  |
--------------------------------------------------------------------------------------------------

DV - Divider pipe (on port 0)
D - Data fetch pipe (on ports 2 and 3)
F - Macro Fusion with the previous instruction occurred
* - instruction micro-ops not bound to a port
^ - Micro Fusion occurred
# - ESP Tracking sync uop was issued
@ - SSE instruction followed an AVX256/AVX512 instruction, dozens of cycles penalty is expected
X - instruction not supported, was not accounted in Analysis

| Num Of   |                    Ports pressure in cycles                         |      |
|  Uops    |  0  - DV    |  1   |  2  -  D    |  3  -  D    |  4   |  5   |  6   |  7   |
-----------------------------------------------------------------------------------------
|   2^     |             | 0.5  | 0.5     0.5 | 0.5     0.5 |      |      | 0.5  |      | add rax, qword ptr [rdi]
|   1      | 0.5         |      |             |             |      | 0.5  |      |      | add rdi, 0x8
|   1*     |             |      |             |             |      |      |      |      | cmp rdi, rdx
|   0*F    |             |      |             |             |      |      |      |      | jnz 0xffffffffffffffee
Total Num Of Uops: 4
```

That's a lot of information to digest.
Let's take it step by step.
First, IACA outputs its own version number and reports the parameters it was called with.
Then, we see a summary of the throughput analysis.
How many cycles each loop iteration takes on average and what the bottleneck is.
We also get an overview of the usage of each port.
At the end, we see a more detailed report about each instruction and how much they used each port.

Interesting to see here is that instructions can also be combined to a single micro-operation.
The comparison `cmp` followed by the conditional jump `jnz` are fused together.
These two types of instructions appear very often one after the other.
It makes sense to handle them in a special way.


# LLVM's Machine Code Analyzer

A similar tool is [LLVM-MCA][mca].
Let's try it out on the same tiny function as before.

{{< highlight "C" >}}
long sum(long *array, long size){
	long result = 0;
	for(long i=0; i<size; ++i){
		__asm volatile("# LLVM-MCA-BEGIN sum-loop");
		result += array[i];
	}
	__asm volatile("# LLVM-MCA-END");
	return result;
}
{{< / highlight >}}

LLVM-MCA does not work on the binary level but on the assembly level.
To mark the code region of our interest, we have to use inline assembly.
At the same positions in the code as before, we place inline assembly statements to insert special comments into the compiler output to the assembler.
They do nothing when the code is assembled to executable code.
Code regions can optionally be named, here "sum-loop".

```
$ gcc -S -O2 -masm=intel sum_mca.c -o sum_mca.s
```

We compile the code only to the assembler code, not further (`-S`).
Additionally, we instruct the compiler to use the intel syntax for the assembly with `-masm=intel`.
IACA uses the intel syntax too.

Now, we just have to call LLVM-MCA with the assembly file.


```
$ llvm-mca sum_mca.s

[0] Code Region - sum-loop

Iterations:        100
Instructions:      400
Total Cycles:      141
Total uOps:        500

Dispatch Width:    4
uOps Per Cycle:    3.55
IPC:               2.84
Block RThroughput: 1.3


Instruction Info:
[1]: #uOps
[2]: Latency
[3]: RThroughput
[4]: MayLoad
[5]: MayStore
[6]: HasSideEffects (U)

[1]    [2]    [3]    [4]    [5]    [6]    Instructions:
 2      6     0.50    *                   add   rax, qword ptr [rdi]
 1      1     0.25                        add   rdi, 8
 1      1     0.25                        cmp   rdi, rdx
 1      1     0.50                        jne   .L3


Resources:
[0]   - HWDivider
[1]   - HWFPDivider
[2]   - HWPort0
[3]   - HWPort1
[4]   - HWPort2
[5]   - HWPort3
[6]   - HWPort4
[7]   - HWPort5
[8]   - HWPort6
[9]   - HWPort7


Resource pressure per iteration:
[0]    [1]    [2]    [3]    [4]    [5]    [6]    [7]    [8]    [9]    
 -      -     1.00   0.99   0.50   0.50    -     1.00   1.01    -     

Resource pressure by instruction:
[0]    [1]    [2]    [3]    [4]    [5]    [6]    [7]    [8]    [9]    Instructions:
 -      -     0.34   0.65   0.50   0.50    -      -     0.01    -     add       rax, qword ptr [rdi]
 -      -      -      -      -      -      -     0.34   0.66    -     add       rdi, 8
 -      -      -     0.34    -      -      -     0.66    -      -     cmp       rdi, rdx
 -      -     0.66    -      -      -      -      -     0.34    -     jne       .L3
```

First, we get the summary of the throughput analysis again.
Next, we have information about each instruction, e.g., _latency_ and _throughput_.
Last, we see the resource allocation, detailing how each instruction got allocated to each port.

The output looks a bit different compared to IACA.
The macro fusion for the loop condition is missing, for example.
I think, they also account different latencies to memory loads and stores.


# Conclusion

I find it very insightful to see what the CPU is actually doing with the sequence of instructions a program is made of.
It makes only sense to do this kind of analysis for very hot inner loops where you want to squeeze some more performance out of.
Especially for vectorized code I usually check at least once what these tools have to say about it.

Even if you are not aiming for top performance, it is interesting to see how it works.
You can also check out the timeline view in LLVM-MCA with the option `-timeline`.
It shows how the pipeline is filled up with various instructions over time.
You can see for yourself how many cycles one iteration requires.

LLVM-MCA is also available on the [Compiler Explorer][godbolt].
On the far right below the compiler options, you can add additional tools.
LLVM-MCA is one of them.
Use the inline assembly as done in the example above.
It will not show up in the assembly view as comments are filtered, but LLVM-MCA will pick it up.
Here is a [link](https://godbolt.org/z/pYNAWY) with the example.

Last but not least, here is a small code example for you to play with and see the effects of unrolling a loop.
Happy analyzing.

{{< highlight "C" >}}
#include <iacaMarks.h>

#ifdef IACA
#  define START IACA_START;
#  define END IACA_END;
#elif defined(MCA)
#  define START __asm volatile("# LLVM-MCA-BEGIN");
#  define END __asm volatile("# LLVM-MCA-END");
#endif


// unrolling without tail handling
// this is just for playing with IACA and LLVM-MCA, not to run this code
long sum(long *array, long size){
	long s=0;
#if UNROLL == 2
	for(long i=0; i<size; i+=2){
START
		s += array[i] + array[i+1];
	}
END
#elif UNROLL == 4
	for(long i=0; i<size; i+=4){
START
		s += array[i] + array[i+1] + array[i+2] + array[i+3];
	}
END
#elif UNROLL == 8
	for(long i=0; i<size; i+=8){
START
		s += array[i] + array[i+1] + array[i+2] + array[i+3]
			+ array[i+4] + array[i+5] + array[i+6] + array[i+7];
	}
END
#else
	for(long i=0; i<size; ++i){
START
		s += array[i];
	}
END
#endif
	return s;
}
{{< / highlight >}}


Compilation for IACA:
```
$ gcc -c -DIACA -DUNROLL=2 -O2 sum.c -o sum.o && iaca sum.o && iaca sum.o
```

Compilation for LLVM-MCA:
```
$ gcc -S -DMCA -DUNROLL=2 -O2 -masm=intel sum.c -o - | llvm-mca
```

(vary the value for `UNROLL` of course)


# References

- [Intel Architecture Code Analyzer][iaca]
- [LLVM Machine Code Analyzer][mca]
- [Compiler Explorer][godbolt]


[iaca]: https://software.intel.com/en-us/articles/intel-architecture-code-analyzer
[mca]: https://llvm.org/docs/CommandGuide/llvm-mca.html
[godbolt]: https://godbolt.org/
