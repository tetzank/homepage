---
title: "Identical Code Folding"
date: 2019-01-07T09:51:00+01:00
---

Even more interesting than removing unused functions is consolidating identical instances of templated functions.
For each template parameter, the compiler generates a new instance.
In case of templated classes, we generate code per template parameter for every member function.
The instances can have identical code, e.g., the member function is independent of the template parameters or the types are semantically equivalent for the applied operations.
Let's see if we can minimize the code explosion by deduplicating code when it is identical.


# Weak Symbols

When a template is instantiated with the same parameters in different .cpp-files, the compiler generates a function with a weak symbol in each object file.
In contrast to other symbols, there can be multiple weak symbols with the exact same name.
The linker assumes that all weak symbols with the same name have identical code.
Therefore, it just picks the first one it encounters when searching for the symbol and copies the code only once into the executable.

Normally, this works like a charm, but it can lead to subtle bugs when templates with the same name (in the same namespace and with the same template parameters) have different implementations.
To make this more clear, let us take a small detour and look at the following example consisting of three source files.

first.cpp:
{{< highlight "C++" >}}
#include <cstdio>

template<int>
void test(){
	puts("test in first.cpp called"); // templated function varies, but has the same name
}

void foo_first(){
	test<0>(); // generate instance, weak symbol with the same name as in second.cpp
}
{{< / highlight >}}

second.cpp (nearly the same as above, `test()` differs slightly in the string):
{{< highlight "C++" >}}
#include <cstdio>

template<int>
void test(){
	puts("test in second.cpp called"); // templated function varies, but has the same name
}

void foo_second(){
	test<0>(); // generate instance, weak symbol with the same name as in first.cpp
}
{{< / highlight >}}

main.cpp:
{{< highlight "C++" >}}
void foo_first();  // forward declare to not need headers
void foo_second(); //

int main(){
	foo_first();  // call both functions
	foo_second(); //
	return 0;
}
{{< / highlight >}}

The program output varies depending on which order the linker gets to know the symbols.

{{< highlight bash >}}
$ g++ main.cpp first.cpp second.cpp -o main && ./main
test in first.cpp called
test in first.cpp called
$ g++ main.cpp second.cpp first.cpp -o main && ./main
test in second.cpp called
test in second.cpp called
{{< / highlight >}}

If we turn on optimizations, the call to `test()` in `foo_first()` and `foo_second()` gets inlined, picking `test()` from the same file.
Hence, we get the expected output.

{{< highlight bash >}}
$ g++ -O2 main.cpp first.cpp second.cpp -o main && ./main
test in first.cpp called
test in second.cpp called
{{< / highlight >}}

As with other classes and functions, never have different implementations with the same name.
Normally, you get a linker error (multiple definitions), but not for templates.


# Consolidating Independent Member Functions

Template instances with the same parameters are fine.
The linker makes sure that there will be only one function in the executable.
But what about different parameters?
Let us look at another small example:

{{< highlight "C++" >}}
// generates a new copy of all member function for each instantiation
template<typename T, int N>
class MyArray{
private:
	int metaData;
	T data[N];

public:
	// identical for all template parameters as it is independent of them
	// 1. How to consolidate and have just one function in the executable?
	[[gnu::noinline]]
	void setMetaData(int meta){
		metaData = meta;
	}

	// depends on T, but generates identical code for types of the same size
	// 2. How to consolidate if identical code was generated?
	[[gnu::noinline]]
	T &operator[](unsigned idx){
		return data[idx];
	}
};

int main(int argc, char *argv[]){
	// instantiate template for two types: int and float, both 4-byte wide
	MyArray<int,  1024> arr_int;
	MyArray<float,1024> arr_float;

	// use setMetaData() for both template types
	arr_int.setMetaData(42);
	arr_float.setMetaData(23);

	// use array access operator for both types
	for(unsigned i=0; i<argc; ++i){
		arr_int[i] = i;
		arr_float[i] = i;
	}
	return 0;
}
{{< / highlight >}}

The program does not compute anything useful.
It just contains a simple template implementing a fixed-size array with the type of the elements and the array size as template parameters, similar to `std::array`.
The member function `setMetaData()` is just a setter independent of the template parameters.
On the other hand, `operator[]()` depends on the template parameter `T`.
Both member functions are marked with the attribute `noinline` which prevents inlining.
Let us assume these functions are big and used at various places throughout the program such that the inliner does not inline the calls.
The code inside of `main` is not important.
It makes use of both member functions and instantiates the template for `int` and `float`.

Let us compile and have a look at the functions in the resulting executable.

{{< highlight txt >}}
$ g++ myarray.cpp -o myarray
$ objdump -tC ./myarray | grep 'MyArray'
000000000000125c  w    F .text  0000000000000023              MyArray<int, 1024>::operator[](unsigned int)
0000000000001244  w    F .text  0000000000000017              MyArray<float, 1024>::setMetaData(int)
000000000000122c  w    F .text  0000000000000017              MyArray<int, 1024>::setMetaData(int)
0000000000001280  w    F .text  0000000000000023              MyArray<float, 1024>::operator[](unsigned int)
{{< / highlight >}}

As we expected, every member function gets instantiated per template parameter.
Now, let us see if optimizations make a difference.

{{< highlight txt >}}
$ g++ -O2 -march=native myarray.cpp -o myarray
$ objdump -tC ./myarray | grep 'MyArray'
00000000000011f0 l     F .text  0000000000000003              MyArray<int, 1024>::setMetaData(int) [clone .isra.0]
00000000000011f0 l     F .text  0000000000000003              MyArray<float, 1024>::setMetaData(int) [clone .isra.1]
0000000000001200  w    F .text  0000000000000008              MyArray<int, 1024>::operator[](unsigned int)
0000000000001210  w    F .text  0000000000000008              MyArray<float, 1024>::operator[](unsigned int)
{{< / highlight >}}

If we look carefully (ignoring the different order), we notice that the first column is now the same for the two instances of `setMetaData()`.
This is the address of the function in the executable.
`-O2` enables `-fipa-icf`[^1] which folds the two instances of `setMetaData()` to one single function, but the symbols remain.
So, we are still littering the symbol table but at least the identical code is shared now.

[^1]: This flag alone is not enough, e.g., `-O1 -fipa-icf` does not work. I do not know which other optimizations need to be enabled which `-O2` enables.

The optimizations did not help for `operator[]()`.
The code is still not shared.
Maybe the assumption, that the code is identical, is not correct.
We can disassemble the code with `objdump -DC` and search for the functions.
Another option is to use `gdb` and its `disassemble` command, but we have to use mangled names.

{{< highlight bash >}}
$ gdb -nx -batch -ex 'disassemble _ZN7MyArrayIfLi1024EEixEj' ./myarray
{{< / highlight >}}

If we enable debug information during compilation, we can also use function signatures[^2].

[^2]: Another option is to use breakpoints: `gdb -nx -batch -ex 'b MyArray<int,1024>::operator[](unsigned int)' -ex 'r' -ex 'disassemble' ./myarray`

{{< highlight bash >}}
$ g++ -g -O2 -march=native myarray.cpp -o myarray
$ gdb -nx -batch -ex 'disassemble MyArray<int,1024>::operator[](unsigned)' ./myarray 
Dump of assembler code for function MyArray<int, 1024>::operator[](unsigned int):
   0x0000000000001200 <+0>:     mov    %esi,%esi
   0x0000000000001202 <+2>:     lea    0x4(%rdi,%rsi,4),%rax
   0x0000000000001207 <+7>:     retq   
End of assembler dump.
$ gdb -nx -batch -ex 'disassemble MyArray<float,1024>::operator[](unsigned)' ./myarray
Dump of assembler code for function MyArray<float, 1024>::operator[](unsigned int):
   0x0000000000001210 <+0>:     mov    %esi,%esi
   0x0000000000001212 <+2>:     lea    0x4(%rdi,%rsi,4),%rax
   0x0000000000001217 <+7>:     retq   
End of assembler dump.
{{< / highlight >}}

Even if you cannot read x86 assembly, you can clearly see that these two functions are identical.


# Consolidating Identical Instances

If we look closely at `MyArray` and how it uses the template parameter `T`, we will notice that it treats it as a black box.
It does not apply any operation on the type.
An array of elements is stored in `data` and accessed by `operator[]()`.
Only the size of `T` makes a difference.
We can reimplement `MyArray` to make this more explicit.

{{< highlight "C++" >}}
template<int size, int amount>
class MyArrayImpl{
private:
	int metaData;
	unsigned char data[size * amount];

public:
	[[gnu::noinline]]
	void setMetaData(int meta){
		metaData = meta;
	}

	[[gnu::noinline]]
	unsigned char *operator[](unsigned idx){
		return &data[idx * size];
	}
};

template<typename T, int N>
class MyArray{
private:
	MyArrayImpl<sizeof(T),N> impl;

public:
	void setMetaData(int meta){
		impl.setMetaData(meta);
	}

	T &operator[](unsigned idx){
		return *(T*)impl[idx];
	}
};
{{< / highlight >}}

`MyArrayImpl` just stores bytes now.
It gets the size of an element and the number of elements passed as template parameters.
`MyArray` acts as a decorator to still provide type safety.
It just forwards all calls to `MyArrayImpl` and reinterprets the stored bytes to the correct type.
The code in `main` does not need to be changed.

{{< highlight txt >}}
$ g++ -O2 -march=native myarray2.cpp -o myarray2
$ objdump -tC ./myarray2 | grep 'MyArray'
00000000000011f0 l     F .text  0000000000000003              MyArrayImpl<4, 1024>::setMetaData(int) [clone .isra.0]
0000000000001200  w    F .text  0000000000000009              MyArrayImpl<4, 1024>::operator[](unsigned int)
{{< / highlight >}}

The member functions in `MyArray` get inlined as they are just forwarding.
`MyArrayImpl` only gets instantiated once with size of 4 as `int` and `float` have the same size.

This is not an ideal solution.
Not every container can treat its elements as black boxes, e.g., containers storing elements in order to provide faster search need to compare elements.
It gets hard to identify groups of types which behave identical considering the applied operations.
An automatic solution would be much appreciated.

The linker to the rescue!
When studying the manpage of GCC and its flag `-fipa-icf`, one gets to know that the gold linker has a similar optimization pass providing ICF.
Time to put it to a test.

{{< highlight bash >}}
$ g++ -O2 -march=native -ffunction-sections -fuse-ld=gold -Wl,--icf=all myarray.cpp -o myarray
{{< / highlight >}}

The linker only works on the level of sections which means we have to put every function in its own section with the flag `-ffunction-sections`.
Then, we have to use `ld.gold` as linker instead of the default `ld`.
The flag `-fuse-ld=gold` changes the linker to gold.
To enable the ICF pass, we pass `--icf=all` to the linker with the `-Wl,<linker-param>` flag.

{{< highlight bash >}}
$ objdump -tC ./myarray | grep 'MyArray'
0000000000000730 l     F .text  0000000000000003              MyArray<int, 1024>::setMetaData(int) [clone .isra.0]
0000000000000730 l     F .text  0000000000000003              MyArray<float, 1024>::setMetaData(int) [clone .isra.1]
0000000000000740  w    F .text  0000000000000008              MyArray<int, 1024>::operator[](unsigned int)
0000000000000740  w    F .text  0000000000000008              MyArray<float, 1024>::operator[](unsigned int)
{{< / highlight >}}

Finally, we get the result we were looking for without modifications of the source code.
Again, the symbol table still contains all instances, but the they point to shared code.

This also works with optimizations turned off for GCC.
The gold linker fixes it all up for us, but it introduces a small size overhead in the executable as we have to put every function in its own section with `-ffunction-sections`.
The gold linker can also be used for clang which does not provide any optimization pass itself for folding identical code.


# Conclusion

GCC with `-O2` can fold member functions which are independent of the template parameters to a single implementation.
This only gets us half the way.
It does not work for member functions which depend on template parameters, but happen to generate identical code.
The gold linker provides an optimization pass to identify identical functions and let them share their code.
An option also available to clang.
