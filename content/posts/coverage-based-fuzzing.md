---
title: "Coverage-Based Fuzzing"
date: 2020-02-08T12:10:24+01:00
---

Software testing is important.
Every developer knows that.
We all try our very very best to keep up with writing tests.
But even a good test coverage is not enough when it comes to writing secure software.
The diversity of inputs and internal states of a program is usually too large.
Some special cases will be forgotten.
Fuzzing tests your software with randomly mutated input data to find bugs and crashes automatically.

Fuzzing is used extensively by security researchers when hunting for bugs which could be potentially exploited.
Software developers should do the same by integrating fuzzing into the testsuite to find vulnerabilities early in the development cycle.
Fuzzing is not difficult.
Multiple open source fuzzing frameworks exist which provide a simple interface and do the heavy lifting for us.
In this post, I will focus on [libFuzzer][libFuzzer] which is integrated into clang/LLVM.


# "Hello, World!" of Fuzzing

Let us begin with a simple example, illustrating how easy it is to define a fuzzing target and execute the fuzzing engine.

{{< highlight "C++" >}}
#include <cstdint>
#include <cstddef>

//                                    pointer to buffer and size of buffer
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
	if (size > 0 && data[0] == 'H'){
		if (size > 1 && data[1] == 'I'){
			if (size > 2 && data[2] == '!'){
				// Hurrah! Input generated to pass all checks. Let's crash!
				__builtin_trap();
			}
		}
	}
	return 0;
}
{{< / highlight >}}

The example has only a single function: `LLVMFuzzerTestOneInput()`.
We are using `extern "C"` to avoid name mangling of the function name.
The function gets called by libFuzzer with a memory buffer of randomly mutated data.
With it, we can perform tests of any functionality we want.
Here, we just want to see if the fuzzer finds its way through a number of branches and generates input data which passes all checks.
The innermost branch contains the statement `__builtin_trap()` which aborts the program execution in an abnormal way, similar to a failed assertion.
It is a direct way to signal the fuzzer about a problem.
You should prefer assertions and exceptions in normal program code.

We compile and run the fuzzing program in the following way.

```
$ clang++ -g -fsanitize=fuzzer hi.cpp -o hi
$ ./hi
```

After only a few iterations, libFuzzer will find an input which causes the program to crash as it executes `__builtin_trap()`.
The input data is written to a 'crash-*' file for you to inspect and start a debugging session with.

This is just a tiny example, but it shows how coverage-based fuzzing finds its ways through branches in your program.


# Combining Fuzzing with Sanitizers

A fuzzer does not know anything about the semantics of your program.
It relies on program crashes and on checks in your program to detect misbehavior.
When you fuzz a piece of code, make sure you enable all available assertions.
Additionally, sanitizers automatically intrument your code with useful checks, detecting errors as early as possible.

In the next example, we combine libFuzzer with address sanitizer (ASAN) which detects common memory errors.

{{< highlight "C++" >}}
#include <cstdint>
#include <cstddef>

// function under test
bool fuzz_me(const uint8_t *data, size_t size){
	return size >= 3
		&& data[0] == 'F'
		&& data[1] == 'U'
		&& data[2] == 'Z'
		&& data[3] == 'Z';
}

// fuzzing target called by fuzzing engine
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size){
	fuzz_me(data, size);
	return 0;
}
{{< / highlight >}}

This time, we have a function under test, called `fuzz_me`, which checks if the passed buffer starts with the characters `"FUZZ"`.
We want to search for hidden bugs in the function which is why we add a fuzzing target calling `fuzz_me`.
Compilation is very similar to the first example.
Next to `fuzzer`, we enable `address` in the `-fsanitize` flag.

```
$ clang++ -g -fsanitize=fuzzer,address fuzz_me.cpp -o fuzz_me
$ ./fuzz_me
```

After a few iterations, libFuzzer finds an input which triggers an assertion added by ASAN, causing the program to terminate.
To understand the error, we start a debugging session with the generated crash file.

```
$ gdb --args ./fuzz_me crash-...
(gdb) b __sanitizer::Die
(gdb) r
```

The fuzzing target is only called with the data in the crash file which caused the issue.
We add a breakpoint on `__sanitizer::Die` which stops program execution right before termination, but after ASAN reports the error.
This allows us to investigate the program state at the time of the error in the debugger.

The bug is a heap-buffer-overflow, reading one byte past the end of the buffer.
`data[3]` accesses the 4<sup>th</sup> element of the buffer `"FUZ"` with only 3 elements.
The size check at the beginning is wrong.

Another useful sanitizer is the undefined behavior sanitizer (UBSAN).
It adds runtime checks to detect when the program relies on undefined behavior which is a bad sign.
The program might work in the current environment, but will most likely misbehave when you change compiler or microarchitecture.

In the next example, we try to implement a left shift in a safe way, avoiding undefined behavior.

{{< todo >}}
 - use input memory to get parameters
 - UBSAN to detect error/undefined behavior which most likely leads to error
 - always crash when sanitizer finds something
 - max_len to speedup mutation
{{< / todo >}}

{{< highlight "C++" >}}
#include <cstdint>
#include <cstddef>

// function under test
uint64_t safe_shift(uint64_t value, uint64_t amount){
	if(amount > 64) return 0;
	return value << amount;
}

// fuzzing target called by fuzzer
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size){
	if(size < sizeof(uint64_t)*2) return 0; // need at least 16 bytes

	uint64_t value  = ((uint64_t*)data)[0]; // extract first integer
	uint64_t amount = ((uint64_t*)data)[1]; // extract second
	return safe_shift(value, amount);
}
{{< / highlight >}}

We extract the two integer parameters of `safe_shift()` from the input buffer with casts.
That's a bit awkward, but it gets the job done.
The compile flags are similar as before.
We add `undefined` to the list of sanitizers and add `-fno-sanitize-recover=all` to always terminate program execution when any sanitizer detects an error.

```
$ clang++ -g -fsanitize=fuzzer,address,undefined -fno-sanitize-recover=all undefined.cpp -o undefined
$ ./undefined -max_len=16
```

We use `-max_len=16` to limit the size of the input buffer to 16 bytes.
Our fuzzing target does not make any use of additional bytes.
Avoiding useless work speeds up the fuzzing process.

Again, after a few iterations, the fuzzer finds a pair of parameters which trigger a check of UBSAN:
"shift exponent 64 is too large for 64-bit type 'uint64_t'."
The check in `safe_shift()` has to be `amount >= 64`.

Shifting all bits from a value is already undefined behavior, e.g., shifting 32 bits left on a 32-bit value or 64 on a 64-bit value.
Many developers expect zero as a result and are surprised that it is undefined in C++.
The reason are divergent semantics on different microarchitectures, e.g., on x86 `x << 64` would result in `x` as the shift instruction has an implicit modulus operation: `x << (64 % 64)`.
By requiring a certain behavior, C++ would impose overhead on platforms with different semantics.
Undefined behavior is the easy way out.

Extracting parameters from the input buffer is a bit tedious.
Fortunately, libFuzzer comes with a helper class `FuzzedDataProvider` to help with structured input.

{{< highlight "C++" >}}
#include <cstdint>
#include <cstddef>

#include <fuzzer/FuzzedDataProvider.h>

// function under test
uint64_t safe_shift(uint64_t value, uint64_t amount){
	if(amount > 64) return 0;
	return value << amount;
}

// fuzzing target called by fuzzer
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size){
	if(size < sizeof(uint64_t)*2) return 0; // need at least 16 bytes

	FuzzedDataProvider fuzdata(data, size);
	uint64_t value  = fuzdata.ConsumeIntegral<uint64_t>(); // first parameter
	uint64_t amount = fuzdata.ConsumeIntegral<uint64_t>(); // second
	return safe_shift(value, amount);
}
{{< / highlight >}}

Looks much better!
Unfortunately, the header file `fuzzer/FuzzedDataProvider.h` might not be included in the packages of your distribution, e.g., the Arch Linux package for "compiler-rt 9" does not include the header file.
Luckily, the header works stand-alone.
You can download it from [here][header].

With `FuzzedDataProvider` it is quite comfortable to generate values of common types like integers and strings from the random input buffer.
It makes our life much easier.


{{< todo >}}

# Structured Fuzzing with Dictionaries

- xml parsing example, needs small code snippet
- fuzzing parsers
- `FuzzedDataProvider` can construct strings from random input, but not following any "grammar"


# Custom Mutator

- zlib compression example

{{< / todo >}}


# Conclusion

I hope, I convinced you that writing a fuzzing target is not that hard.
Fuzzing should become a common tool in every programmers toolset to improve the quality of the whole software ecosystem.
For important open source projects, [OSS-Fuzz][ossfuzz] from Google offers to run the fuzzing process for you.
You just have to provide the fuzzing targets.

{{< todo >}}
Let's strive for a bug-free world!
{{< / todo >}}


# References

- [LibFuzzer][libFuzzer]
- [Fuzzing documentation by Google][googlefuzzing]
- [OSS-Fuzz project][ossfuzz]
- [Informative presentation about fuzzing][talk]


[talk]: https://www.youtube.com/watch?v=x0FQkAPokfE
[libFuzzer]: http://llvm.org/docs/LibFuzzer.html
[googlefuzzing]: https://github.com/google/fuzzing/blob/master/docs
[header]: https://raw.githubusercontent.com/llvm/llvm-project/master/compiler-rt/include/fuzzer/FuzzedDataProvider.h
[ossfuzz]: https://google.github.io/oss-fuzz/
