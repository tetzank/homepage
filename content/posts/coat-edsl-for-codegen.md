---
title: "COAT: EDSL for Codegen"
date: 2019-09-12T20:40:01+02:00
---

Code specialization has a huge impact on performance.
Tailored code takes advantage of the knowledge about the involved data types and operations.
In C++, we can instruct the compiler to generate specialized code at compile-time with the help of template metaprogramming and constant expressions.
However, constant evaluation is limited as it cannot leverage runtime information.
Just-in-time compilations lifts this limitation by enabling programs to generate code at runtime.
In this post, I will present a header-only library providing abstract types and control flow abstractions to make code generation at runtime easier to use.

__Disclaimer: This project is still in its early alpha stage. The code is available on [github][repo].__


# Partial Evaluation

In programming language theory there is the interesting concept of partial evaluation.
Let's assume that parts of the input state of a program are known at a certain point in time.
Hence, some of the expressions and statements for which we know the inputs can be evaluated and the whole program transformed into a specialized form for this partial input.
The specialized program ought to be faster as we already evaluated parts of its expressions.

C++ provides language features to support this concept like constant expressions.
Constant expressions are evaluated during compilation when all inputs are constants.
Only their results leak to the runtime of the program.
The point in time for the transformation is restricted to compile-time.
There is no support in the language to generate code at runtime.

Depending on the task of the program, code specialization at runtime can be very beneficial.
Let's assume for a second that your program takes complex tasks as an input.
Each task consists of a sequence of operations which must be applied to a large number of data items.
Instead of interpreting each operation one after the other for each data item, it would be much better to generate a single function which applies all operations of the task at hand and just call this function for each data item.
I'm a database researcher.
Naturally, I'm thinking about database queries here.

Luckily, compiler frameworks like LLVM are built in a modular way that the code generation and optimization part can be used as a library which allows an application to use them at runtime.
Nevertheless, it is far from easy to integrate just-in-time compilation into an application.
The compiler APIs are complex and generating LLVM IR, the intermediate language/representation of LLVM, is tedious and error-prone.
LLVM IR is like an architecture agnostic assembler language which we have to create one instruction after the other with complex API calls.
All in all, it is similar to writing assembly by hand.

That is clearly not the most productive way to spend your time which is why I added more and more abstractions on top of the compiler API and ended up with an embedded domain specific language (EDSL) to make code generation easier to use.


# Embedded Domain Specific Language (EDSL)

An EDSL is part of the source code it is embedded in (host language).
The syntax and operators of the host language are customized on special data types to express the DSL inside the host language.
Operator overloading in C++ comes in handy here.

The obvious advantage is the tight integration into the surrounding source code.
There are no language boundaries, no parsers or other tools.
It follows all the syntax rules of the host language which is a big plus as developers are familiar with it, but it can also be a big limitation as we cannot create more fitting abstractions in the syntax for the specific domain we are dealing with.
Furthermore, for the untrained eye, it might be hard to distinguish which values have customized operators and therefore a different semantic than the standard operators.
Everything has its learning curve.

The most compelling advantage for me is type-safety at compile-time.
The EDSL is integrated and therefore compiled together with all the other source code.
With compile-time checks, we can verify that the EDSL is well-formed.


# Codegen Abstract Types (COAT)

COAT is an EDSL for C++ which makes just-in-time code generation easier.
It provides data types and control flow abstractions to express the code we want to generate.
In a nutshell, instead of evaluating an operator, we write equivalent instructions through the compiler API to the generated function.
The instruction is chosen considering the types of the operands.

An example is better than a thousand words.

{{< highlight "C++" >}}
#include <cstdio>
#include <vector>
#include <numeric>

#include <coat/Function.h>
#include <coat/ControlFlow.h>


int main(){
	// generate some data
	std::vector<uint64_t> data(1024);
	std::iota(data.begin(), data.end(), 0);

	// initialize backend, AsmJit in this case
	coat::runtimeasmjit asmrt;
	// signature of the generated function
	using func_t = uint64_t (*)(uint64_t *data, uint64_t size);
	// context object representing the generated function
	coat::Function<coat::runtimeasmjit,func_t> fn(&asmrt);
	// start of the EDSL code describing the code of the generated function
	{
		// get function arguments as "meta-variables"
		auto [data,size] = fn.getArguments("data", "size");

		// "meta-variable" for sum
		coat::Value sum(fn, uint64_t(0), "sum");
		// "meta-variable" for past-the-end pointer
		auto end = data + size;
		// loop over all elements
		coat::for_each(fn, data, end, [&](auto &element){
			// add each element to the sum
			sum += element;
		});
		// specify return value
		coat::ret(fn, sum);
	}
	// finalize code generation and get function pointer to the generated function
	func_t foo = fn.finalize(&asmrt);

	// execute the generated function
	uint64_t result = foo(data.data(), data.size());

	// print result
	uint64_t expected = std::accumulate(data.begin(), data.end(), 0);
	printf("result: %lu; expected: %lu\n", result, expected);

	return 0;
}
{{< / highlight >}}

In this example, we generate a function which calculates the sum of a vector.
This is hardly a useful application of just-in-time code generation, but it is small enough to show the full program, and it gets the main idea of COAT across.

After generating some basic data, we initialize one of the compiler backends of COAT.
At the moment, COAT supports two backends: [AsmJit][asmjit] and [LLVM][llvm].
Next, we define the signature of the function we want to generate with `using`.
The data array is passed with a pointer and size pair to the function which in turn returns the sum of all elements.
Finally, we create a `coat::Function` object with the compiler backend and function signature as template parameters.
It will be the context object representing the generated function.
Internally, it prepares the compiler backend to generate the function with the requested signature.

Afterwards, we start writing the content of the generated function.
We open an artificial block scope to emphasize the code segment with the EDSL types.
All objects in this scope are meta objects which generate code.
They are used to describe the code to be generated in a natural and readable way.

At the beginning of the scope, we get the arguments from the function context.
The types and number of arguments are automatically deduced from the function signature.
Next, we create a variable for the sum and initialize it to zero, followed by the definition of the past-the-end pointer for the array.
Then, we use the loop construct `coat::for_each` which generates a loop to iterate over all elements between two pointers, similar to `std::for_each`.
The passed lambda acts as the loop body and sums up all the elements.
Finally, the sum variable is specified as return value with `coat::ret`.

All in all, the EDSL looks pretty much like normal C++ code.
The details of the compiler API are hidden behind the abstraction of these types.
The code is much easier to read and maintain than the sea of complex API calls one usually has with just-in-time compilers.

We can mix C++ code and EDSL code in any way we see fit, e.g., we could pre-calculate some values during code generation and add them as constants to the generated code.
This is analogue to constant expressions pre-calculating at compile-time and adding constants to the runtime code.
One other common case is the conditional generation of code.
Depending on the program state or some input, we generate different code fragments.
The condition can be ordinary C++ code.

{{< todo >}}
- example with partial evaluation where C++ code and EDSL are interleaved
{{< / todo >}}


# Control Flow Abstractions

In the example, we already saw `coat::for_each` in action.
It takes two pointers and increments in each loop iteration the first pointer until it is equal to the second pointer.
It generates code similar to a for-loop.

For other loop and branch constructs of C++, COAT provides similar abstractions.
The following table summarizes all the control flow abstractions and relate them to the equivalent C++ code.

<table style="width: 100%; border-spacing: .5em 1em;">
<tr><th>C++</th><th>COAT</th></tr>
<tr><td>
{{< highlight "C++" >}}
if( condition ){
	then_branch
}
{{< / highlight >}}
</td><td>
{{< highlight "C++" >}}
coat::if_then(coat::Function&, condition, [&]{
	then_branch
});
{{< / highlight >}}
</td></tr>
<tr><td>
{{< highlight "C++" >}}
if( condition ){
	then_branch
}else{
	else_branch
}
{{< / highlight >}}
</td><td>
{{< highlight "C++" >}}
coat::if_then_else(coat::Function&, condition, [&]{
	then_branch
}, [&]{
	else_branch
});
{{< / highlight >}}
</td></tr>
<tr><td>
{{< highlight "C++" >}}
while( condition ){
	loop_body
}
{{< / highlight >}}
</td><td>
{{< highlight "C++" >}}
coat::loop_while(coat::Function&, condition, [&]{
	loop_body
});
{{< / highlight >}}
</td></tr>
<tr><td>
{{< highlight "C++" >}}
do {
	loop_body
} while( condition );
{{< / highlight >}}
</td><td>
{{< highlight "C++" >}}
coat::do_while(coat::Function&, [&]{
	loop_body
}, condition );
{{< / highlight >}}
</td></tr>
</table>

To make this clear again, the code on the right is only executed once, not iterating in case of a loop, and generates machine code in the end which is equivalent to the C++ code on the left.
We are describing the generated code.
When we call the generated function, the loops and branches will be executed.

{{< todo >}}
- structs: member access, extend wrapper type with functionality
{{< / todo >}}

# Implementation Details

In this section, I will briefly talk about the two backends COAT currently supports: AsmJit and LLVM.
And also give some details about the implementation of each backend.


__AsmJit backend__

[AsmJit][asmjit] is a C++ library providing a just-in-time assembler.
It supports x86 assembly with all of its extensions.
The API is quite simple.
After a few initializations, we can start generating x86 instructions one after the other.
We are literally writing assembly with library calls.

The generated code will be executed as-is.
There are no optimization passes cleaning up the code.
It's just an assembler.
When using the "Compiler" backend of AsmJit, we can use virtual registers which are unlimited in number.
A register allocator pass will map virtual register to physical registers when finalizing the machine code.

The AsmJit backend of COAT makes heavy use of virtual registers.
A `coat::Value` representing a variable in the generated code is basically just a wrapper around a virtual register.
All arithmetic operators are customized to generate instructions using the virtual registers of the operands.

The generation is done immediately in the overloaded operator which means that temporaries used in nested expressions cannot be eliminated.
Some SIMD libraries use expression trees to capture the whole expression and eliminate unnecessary temporaries.
This is not done here.
The expressions are mapped 1:1 to the corresponding x86 instructions.

The compilation latency of AsmJit is very low.
If you want to generate a function as fast as possible, this is the right backend for you.
The efficiency of the generated code is up to you.
You have to write efficient code.


__LLVM backend__

[LLVM][llvm] is a modular compiler framework providing ahead-of-time and just-in-time compiler support.
Clang is the C++ frontend of LLVM.
The API is quite complex as LLVM supports a lot of different compilation modes.

LLVM IR is the intermediate representation of the program code inside of LLVM.
It is the common currency between most components.
To keep compilation latency at a minimum, the LLVM backend of COAT generates LLVM IR instructions and passes them to the LLVM libraries, avoiding the C++ frontend.
Optionally, various optimization passes can be applied to the generated LLVM IR, e.g., removing unnecessary temporaries.
Finally, machine code can be generated with the help of various backends supporting a multitude of microarchitectures.

LLVM IR is in SSA form (static single assignment) which means that every value is immutable once created.
This is an essential property for optimization passes but a bit inpractical for code generation.
Similar to a C++ frontend, we can work around this limitation by storing values in memory, e.g., on the stack.
Every overloaded operator must first load the value from memory, apply its operation and store the result back to memory.
Optimization passes make this more efficient by storing values in registers where possible.

The compilation latency of LLVM is much bigger compared to AsmJit, but LLVM provides optimization passes which can make a big difference in the runtime of the generated function.


# Similar Projects

A somewhat similar recent approach is [ClangJIT](https://arxiv.org/abs/1904.08555).
To put it simply, it defers instantiation of annotated templates from compile-time to runtime.
Therefore, it integrates nicely into C++ source code.
We are reusing existing language features like templates and the compiler does all the heavy lifting for us.
No need for special libraries.

The disadvantage is obviously the required compiler support which is currently limited to this fork of Clang.
COAT works with any modern C++ compiler.
Furthermore, compilation latency suffers from the fact that the application has to carry the full compiler along, including the C++ frontend instantiating the template at runtime.
In COAT, you can choose the backend, and with the AsmJit backend compilation latency is very low.

Another approach to simplify JIT compilation is [Easy::Jit](https://github.com/jmmartinez/easy-just-in-time).
Like ClangJIT, it relies heavily on the compiler to do the magic.
A plugin for Clang is provided to inject an "optimization" pass which additionally stores LLVM IR of annotated functions in the executable.
The LLVM IR is later used by the JIT compilation.

The API is very simple, just a single function call.
It relies on compiler assistance which makes it easy to use but also results in a high compilation latency.


# Conclusion

The source code is available in a [github repository][repo].
The project is still in its early alpha stage with a lot of limitations, e.g., debugging support is completely missing at the moment.
With the help of others, I hope, it can become a useful tool for C++ developers.

In the next post, I will present a more comprehensive example for just-in-time compilation.
We will look at modern relational databases and how they make use of code generation for query execution.


# References

- [Source code repository][repo]
- [AsmJit library][asmjit]
- [LLVM compiler framework][llvm]


[repo]: https://github.com/tetzank/coat
[asmjit]: https://asmjit.com/
[llvm]: https://llvm.org/
