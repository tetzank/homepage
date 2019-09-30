---
title: "Codegen in Databases"
date: 2019-09-18T15:25:08+02:00
---

Just-in-time compilation is usually associated with managed languages like Java and C#, or scripting languages like Javascript.
As detailed in the previous post, many other applications benefit from ad hoc code generation as well.
This post is a tutorial on code generation in relational databases.
Relational databases are commonly accessed with the standard query language SQL.
The query optimizer generates an optimized query plan and passes it to the query execution engine for processing which in modern systems generates machine code for faster execution.


# SIGMOD Programming Contest 2018

SIGMOD is one of the most popular database conferences.
Each year, a programming contest is held, and the top finalists can present their implementations at the venue.
It is all about performance.
The submissions are ranked by their evaluation time.
The winner with the fastest program usually organizes the contest in the following year.

In 2018, [the task](http://sigmod18contest.db.in.tum.de/task.shtml) was to write a prototype of a query execution engine.
The queries are quite simple, consisting only of joins with an equivalence predicate (equi-joins) and simple filters (single predicate, no complex expressions).
Furthermore, there is only one data type: an integer of 8 bytes.
This limits the number of operators the execution engine must support considerably which makes it a nice task for a programming contest, and a tutorial such as this one.

The queries are in an easy to parse text format. Let's have a look at one example query.

Example query: `0 2 4|0.1=1.2&1.0=2.1&0.1>3000|0.0 1.1`

There are three parts:

1. The list of relations participating in the query.
2. All equi-joins and filters combined in one conjunction.
3. The list of columns to project on which implicitly sums up all elements and just returns a single tuple with the sums for each projected column.

In SQL, it would look like the following query.

{{< highlight "SQL" >}}
SELECT SUM("0".c0), SUM("1".c1)
FROM r0 "0", r2 "1", r4 "2"
WHERE "0".c1="1".c2 and "1".c0="2".c1 and "0".c1>3000
{{< / highlight >}}

The implicit sum in the projection is quite interesting.
It means that the result set is always just a single tuple.
If we can keep the amount of intermediate results as low as possible before condensing it to a single list of sums in the projections, we are golden.
With code generation we can avoid most of the intermediate results by just keeping track of the position in the table (row id).
As we will see in the next section, additional optimizations eliminate the overhead inherent to interpretation, resulting in a small amount of instructions doing only the necessary work.

Code generation is perceived by a lot of developers as very complex and hard to grasp, if one is not a compiler developer.
I argue, it is not that hard if one gets into the right mindset and uses the right tools.
I hope that you will agree with me at the end of this tutorial.


# Data-centric Execution Pipeline

We follow the approach of data-centric code generation introduced in [HyPer](http://hyper-db.com/) with the paper "Efficiently Compiling Efficient Query Plans for Modern Hardware" by Thomas Neumann.
Check out the list of publications on their website if you want to get deeper into the topic.

We focus purely on the query execution engine in this tutorial.
A database does a lot of optimizations beforehand during query planning.
The order in which operators are executed, especially the join order, has a large impact on the query execution time.
For simplicity, we assume that this is already done and returns a left deep query plan with one long operator pipeline from a scan all the way to the projection.

<svg version="1.1" width="430" height="300" xmlns="http://www.w3.org/2000/svg">
<g text-anchor="middle">
<g font-weight="bold" text-decoration="underline">
<text x="120" y="15">Left Deep Query Plan</text>
<text x="350" y="15">Operator Pipeline</text>
</g>
<text x="50" y="300">R</text>
<text x="50" y="240">&sigma;<tspan dy="3" font-size="smaller">R<tspan dy="3" font-size="smaller">1</tspan><tspan dy="-3">&gt;3000</tspan></tspan></text>
<text x="100" y="180">&bowtie;<tspan dy="3" font-size="smaller">R<tspan dy="3" font-size="smaller">1</tspan><tspan dy="-3">=S</tspan><tspan dy="3" font-size="smaller">2</tspan></tspan></text>
<text x="150" y="120">&bowtie;<tspan dy="3" font-size="smaller">S<tspan dy="3" font-size="smaller">0</tspan><tspan dy="-3">=T</tspan><tspan dy="3" font-size="smaller">1</tspan></tspan></text>
<text x="150" y="60">&pi;<tspan dy="3" font-size="smaller">R<tspan dy="3" font-size="smaller">0</tspan><tspan dy="-3">,S</tspan><tspan dy="3" font-size="smaller">1</tspan></tspan></text>
<text x="150" y="240">S</text>
<text x="200" y="180">T</text>
<path d="M50 280 v-30 m10 -25 l30 -35 m20 -25 l30 -35 M150 100 v-30" stroke="red" stroke-width="4" fill="none"/>
<path d="M140 220 l-25 -25 m-5 5 l10 -10 M190 160 l-25 -25 m-5 5 l10 -10" stroke="black" stroke-width="2" fill="none"/>
<text x="350" y="300">Scan</text>
<text x="350" y="240">Filter</text>
<text x="350" y="180">Hashjoin</text>
<text x="350" y="120">Hashjoin</text>
<text x="350" y="60">Projection/Sum</text>
<path id="darrow" d="M350 280 v-30 m4 0 v30 m5 -25 l-7 -7 l-7 7" stroke="red" stroke-width="2" fill="none"/>
<use href="#darrow" transform="translate(0 -60)" />
<use href="#darrow" transform="translate(0 -120)" />
<use href="#darrow" transform="translate(0 -180)" />
</g>
</svg>

The figure on the left is a left deep query plan of the example query.
For clarity, the participating tables were renamed to R, S and T.
The long operator pipeline is marked in red.
The sequence of operators in the pipeline is illustrated on the right, from the bottom to the top.
The results of an operator are pushed into the next operator until the projection is reached where the results are aggregated into a sum per projected column, here R<sub>0</sub> and S<sub>1</sub>.

Our goal is to generate code to execute the pipeline.
We gracefully ignore the build phase of the hashjoins constructing the hashtables and assume that it is already taken care of.
The following handwritten C++ code shall be the template for the code generation, illustrating how the operators are merged together into a single function.

{{< highlight "C++" >}}
// result variables for projection
uint32_t num=0, p0=0, p1=0;

// scan of R, loop over all row ids
for(uint32_t R_index=0; R_index<R.size; ++R_index){

	// filter on R1
	if(R1[R_index] > 3000){

		// hashjoin R1=S2, prepared hashtable of S2 containing row ids of S
		auto [it,itend] = HT_S2.equal_range(R1[R_index]);
		// loop over all join partners
		for(; it!=itend; it++){
			// set row id for this join partner
			uint32_t S_index = *it;

			// hash join S0=T1
			auto [it2,itend2] = HT_T1.equal_range(S0[S_index]);
			for(; it2!=itend2; it2++){
				// row id is unused, semi-join
				uint32_t T_index = *it2;

				// projection
				++num;             // number of rows in aggregate
				p0 += R0[R_index]; // first sum
				p1 += S1[S_index]; // second sum
			}
		}
	}
}
{{< / highlight >}}

At the beginning, we declare and initialize the result variables of the projection.
The first operator is a scan of the table R.
As we want to keep the intermediate results as low as possible, we simply loop over all row ids of the table and only access the elements of the table when needed.

Next, we apply the filter on R<sub>1</sub>.
The tables are in columnar storage, i.e., a table is a collection of columns.
Each column is an array of integers.
We access the column R<sub>1</sub> at the row id coming from the scan and compare it with the filter predicate.

Afterwards, we join table R and S when R<sub>1</sub> is equal to S<sub>2</sub>.
The hashtable `HT_S2` was constructed beforehand in the build phase.
It is a multimap containing for each distinct value of S<sub>2</sub> a list of row ids representing the positions of the value in the column.
We probe the hashtable with the current value of R<sub>1</sub> to get the row ids of the join partners in S.
We loop over all join partners to consider them one after the other.
The next hashjoin works in the same manner.

Finally, in the most inner loop, we have the code for the implicit sum in the projection.
We simply access each projected column with the current row id of the corresponding table and add it to the sum.
Additionally, we keep track of the number of values added to the sum to distinguish an empty result set from a sum which is equal to zero.

As we can see, the operators boil down to just a few statements and are seamlessly nested into each other.
There are no operator boundaries leading to any call overhead.
The code is completely inlined.
Furthermore, the usage of row ids to avoid intermediate results feels quite natural.
The code is clean and easy to follow.

In the next section, we will discuss some variants of code generation before jumping into the actual implementation.


# Variants of Code Generation

There are multiple ways to generate code in a database.
It depends on what language or tool we want to target which in turn does the heavy lifting of generating the native machine code of the CPU.
The following figure summarizes the various alternatives.

<svg version="1.1" width="480" height="440" xmlns="http://www.w3.org/2000/svg">
<defs>
<marker id='head' orient="auto" markerWidth='12' markerHeight='8' refX='0.1' refY='4'>
<path d='M0 0 V8 L12 4 Z' fill="black"/>
</marker>
</defs>
<g text-anchor="middle">
<g transform="translate(50,20)">
<rect x="5" y="5" width="110" height="160" stroke="black" stroke-dasharray="4" fill="none"/>
<rect x="20" y="10" width="80" height="30" stroke="black" fill="none"/>
<rect x="10" y="110" width="100" height="50" stroke="black" fill="none"/>
<text x="60" y="30">Planning</text>
<text x="60" y="130">Code</text>
<text x="60" y="150">Generation</text>
<path d="M60 40 v60" marker-end="url(#head)" stroke="black" stroke-width="1" fill="none" />
<text x="80" y="80">Plan</text>
</g>
<text x="25" y="35">Query</text>
<path d="M5 45 H60" marker-end="url(#head)" stroke="black" stroke-width="1" fill="none"/>
<text x="110" y="15">Database</text>
</g>
<g text-anchor="middle" transform="translate(280,0)">
<g transform="translate(5,30)">
<rect x="0" y="0" width="180" height="65" stroke="black" fill="none"/>
<text x="90" y="20">Front End</text>
<text x="30" y="50">C++</text>
<text x="80" y="50">Java</text>
<text x="140" y="50">Fortran</text>
<path d="M5 60 h170 v-30 h-170 Z m50 0 v-30 m50 0 v30" stroke="black" fill="none"/>
</g>
<g transform="translate(5,150)">
<rect x="0" y="0" width="180" height="95" stroke="black" fill="none"/>
<text x="90" y="20">Middle End</text>
<text x="90" y="50">Program Analysis</text>
<text x="90" y="80">Optimizations</text>
<path d="M5 90 h170 v-60 h-170 v30 h170 M5 90 v-30" stroke="black" fill="none"/>
</g>
<g transform="translate(5,300)">
<rect x="0" y="0" width="180" height="95" stroke="black" fill="none"/>
<text x="90" y="20">Back End</text>
<text x="30" y="50">x86</text>
<text x="80" y="50">MIPS</text>
<text x="140" y="50">SPARC</text>
<text x="90" y="80">Register Allocator</text>
<path d="M5 60 h170 v-30 h-170 Z m50 0 v-30 m50 0 v30 M5 60 v30 h170 v-30" stroke="black" fill="none"/>
</g>
<text x="90" y="15">Compiler</text>
<rect x="0" y="25" width="190" height="375" stroke="black" stroke-dasharray="4" fill="none"/>
<path d="M95 95 v45" marker-end="url(#head)" stroke="black" fill="none"/>
<text x="110" y="130">IR</text>
<path d="M95 245 v45" marker-end="url(#head)" stroke="black" fill="none"/>
<text x="110" y="280">IR</text>
<path d="M95 395 v35" marker-end="url(#head)" stroke="black" fill="none"/>
<text x="150" y="420">native code</text>
</g>
<g stroke="black" fill="none" marker-end="url(#head)">
<path id="a" d="M160 130 C220 110,250 0,300 24"/>
<path id="b" d="M160 146 Q240 110,300 144"/>
<path id="c" d="M160 162 C250 170,250 280,300 296"/>
<path id="d" d="M160 180 C210 200,200 375,287 375"/>
</g>
<g text-anchor="middle">
<text dy="-2"><textPath href="#a" startOffset="50%">a) source code</textPath></text>
<text dy="-2"><textPath href="#b" startOffset="50%">b) IR</textPath></text>
<text dy="-2"><textPath href="#c" startOffset="50%">c) IR</textPath></text>
<text dy="-2"><textPath href="#d" startOffset="50%">d) assembly</textPath></text>
</g>
</svg>

Modern compiler frameworks like LLVM are modular, consisting of three main components:
language _front ends_ parsing source code, the _middle end_ containing most optimizations, and multiple _back ends_ supporting various microarchitectures.
Each component can be targeted by the code generation of the database.

__a)__
We can generate source code of a high-level programming language like C++.
The result would look similar to the example source code in the previous section.
The main advantage is ease-of-use.
Developers are familiar with source code.
It is not so complex to generate source code for each operator in a nested fashion.
The main disadvantage is the high compilation latency.
The generated source code must go through all the stages of the compiler to be ready for execution.
For short running queries, the compilation takes way longer than the query evaluation, even when interpreting.

__b)__
We can reduce the overhead of the compilation by skipping the language _front end_ and passing the intermediate representation (IR) directly to the compiler.
We still leverage all the optimizations the compiler has in store for us, but IR is quite complex to generate.
The IR in the LLVM compiler framework is similar to an architecture agnostic assembly language in a special format which has useful properties for the optimization passes but is not easy to write.

__c)__
We can shed off even more compilation overhead by sending the IR to the _back end_ and doing an unoptimized build.
Depending on the query, this can have a large negative influence on the execution time of the generated code.

__d)__
The last variant skips most of the compiler stages by generating assembly instructions of the target microarchitecture.
It only leverages the register allocator of the _back end_ which assigns virtual registers to physical registers.
The use of virtual registers makes the generation of nested code fragments much easier.
Nevertheless, we are writing assembly which is a tedious and error-prone process, and not portable at all.

To make our life easier, we will use COAT, an EDSL for C++ which simplifies the implementation of code generation.
For details, see the [previous post](/posts/coat-edsl-for-codegen/) introducing it.
The variants b) and c) are available with COAT's LLVM backend, variant d) with the AsmJit backend.
The complexity is hidden from sight behind the EDSL.
We reap the benefits without sacrificing the ease-of-use.


# Morsel-Driven Parallelism

The parallelization scheme we will use is very simple: we partition the input of the pipeline in equal-sized batches, called morsels.
Each morsel is independent of any other morsel and is executed in parallel without any synchronization.
When a morsel is fully processed, the resulting sums are added atomically to global sums which store the result of the whole query.

This parallelization scheme is straightforward and only changes the code generation of the _scan_ operator slightly, as we will see in the next section.


# Operator Implementations

In this section, I will explain how the code generation is done in the operators.
The whole project is available in a [git repository on github](https://github.com/tetzank/sigmod18contest).
You can check out the code to see all the details.

To recap, the programming contest of 2018 requires only a few operators: _scan_, _filter_, _equi-join_ and _projection/sum_.
Moreover, there is only one data type we have to support.
Everything is an integer.
A pipeline is a sequence of operators.
For this workload, the pipeline always starts with a _scan_ and ends in a _projection_.

For simplicity, we link all operators together in a linked list.
Each operator, except _projection_ of course, calls the next operator in the list at the code position where it should place its code.
Code can be placed before and after the code of the nested operators.

Here's a simple abstract class we use as the base class for all operators.

{{< highlight "C++" >}}
class Operator{
protected:
	Operator *next=nullptr;

public:
	Operator(){}
	virtual ~Operator(){
		delete next;
	}

	void setNext(Operator *next){
		this->next = next;
	}

	// code generation with coat, for each backend, chosen at runtime
	virtual void codegen(Fn_asmjit&, CodegenContext<Fn_asmjit>&)=0;
	virtual void codegen(Fn_llvmjit&, CodegenContext<Fn_llvmjit>&)=0;
};
{{< / highlight >}}

There are two pure virtual member functions `codegen`: one for the LLVM backend and one for the AsmJit backend.
`CodegenContext` is a struct holding information each operator can use when generating code, e.g., a list of row ids, one for each participating table.
{{< todo >}}
Most operators need access to the row id of the currently processed row in the table the operator works on.
{{< / todo >}}

The _scan_ operator is called first.
It iterates over all row ids in the morsel.

{{< highlight "C++" >}}
class ScanOperator final : public Operator{
private:
	template<class Fn>
	void codegen_impl(Fn &fn, CodegenContext<Fn> &ctx){
		// get lower bound of morsel
		// do not make a copy, just take the virtual register from arguments
		ctx.rowids[0] = std::move(std::get<0>(ctx.arguments));
		// get upper bound of morsel
		auto &upper = std::get<1>(ctx.arguments);
		// iterate over morsel
		coat::do_while(fn, [&]{
			// call next operator
			next->codegen(fn, ctx);
			// next row id
			++ctx.rowids[0];
		}, ctx.rowids[0] < upper);
	}

public:
	ScanOperator(const Relation &relation) {}

	void codegen(Fn_asmjit &fn, CodegenContext<Fn_asmjit> &ctx) override { codegen_impl(fn, ctx); }
	void codegen(Fn_llvmjit &fn, CodegenContext<Fn_llvmjit> &ctx) override { codegen_impl(fn, ctx); }
};
{{< / highlight >}}

Both COAT backends share the same operator implementation in `codegen_impl`.
All other operators do the same which is why only `codegen_impl` is shown from now on.

First, we get the start value of the morsel from the first argument and reuse it as the value representing the row id of the first table.
Next, we get the upper bound of the morsel from the second argument.
Finally, we use `coat::do_while()` to generate a loop which will iterate as long as there are row ids left in the morsel.

The lambda is used as the loop body.
It calls `codegen` on the next operator to insert the nested code into the loop body.
Finally, it increments the row id of the morsel.

That's all there is to a _scan_ operator.
The _filter_ operator is straightforward as well.

{{< highlight "C++" >}}
template<class Fn>
void codegen_impl(Fn &fn, CodegenContext<Fn> &ctx){
	// read from column, depends on column type
	auto val = loadValue(fn, column, ctx.rowids[relid]);
	// conditionally generate code depending on comparison type
	switch(comparison){
		case Filter::Comparison::Less: {
			coat::if_then(fn, val < constant, [&]{
				next->codegen(fn, ctx);
			});
			break;
		}
		case Filter::Comparison::Greater: {
			coat::if_then(fn, val > constant, [&]{
				next->codegen(fn, ctx);
			});
			break;
		}
		case Filter::Comparison::Equal: {
			coat::if_then(fn, val == constant, [&]{
				next->codegen(fn, ctx);
			});
			break;
		}
	}
}
{{< / highlight >}}

First, we read the value from the column of the table we filter at the position of the current row id of this table.
Next, we conditionally generate code for the different comparison types.
For each comparison type, we call `coat::if_then()` with the corresponding condition.
The lambda we pass is the then-branch.
It just calls the next operator to insert the code inside the then-branch.

For reading the value, we used the helper function `loadValue`.
It generates slightly different code to fetch the value depending on the type of the column.
During load, columns with small values got moved to a denser 32-bit or 16-bit representation instead of the initial 64-bit.
Thus, we have three types of columns to handle.
More advanced compression schemes can be added as well.

{{< highlight "C++" >}}
template<class Fn, class CC>
coat::Value<CC,uint64_t> loadValue(Fn &fn, const column_t &col, coat::Value<CC,uint64_t> &idx){
	coat::Value<CC, uint64_t> loaded(fn, "loaded");
	switch(col.index()){
		case 0: {
			auto vr_col = fn.embedValue(std::get<uint64_t*>(col), "col");
			// fetch 64 bit value from column
			loaded = vr_col[idx];
			break;
		}
		case 1: {
			auto vr_col = fn.embedValue(std::get<uint32_t*>(col), "col");
			// fetch 32 bit value from column and extend to 64 bit
			loaded.widen(vr_col[idx]);
			break;
		}
		case 2: {
			auto vr_col = fn.embedValue(std::get<uint16_t*>(col), "col");
			// fetch 16 bit value from column and extend to 64 bit
			loaded.widen(vr_col[idx]);
			break;
		}

		default:
			fprintf(stderr, "unknown type in column_t: %lu\n", col.index());
			abort();
	}
	return loaded;
}
{{< / highlight >}}


Similarly, the _equi-join_ operator is quite simple.
Most of the things happen in the "hash table" which is actually a concise array table (similar to CSR for the graph people among us).

{{< highlight "C++" >}}
template<class Fn>
void codegen_impl(Fn &fn, CodegenContext<Fn> &ctx){
	// fetch value from probed column
	auto val = loadValue(fn, probeColumn, ctx.rowids[probeRelation]);
	// embed pointer to hashtable in the generated code
	auto ht = fn.embedValue(hashtable, "hashtable");
	// iterate over all join partners
	ht.iterate(val, [&](auto &ele){
		// set rowid of joined relation
		ctx.rowids[buildRelation] = ele;
		next->codegen(fn, ctx);
	});
}
{{<  / highlight >}}

First, we fetch the value from the probed column.
Next, we embed the pointer to the "hash table" in the generated code and get the `coat::Struct` object `ht` back, initialized with the pointer address.
The address of the pointer is stored in the generated code as an immediate value.
`coat::Struct` is a wrapper for a pointer to a struct/class and provides access to member variables when they are _marked_ in a special way.

{{< highlight "C++" >}}
template<typename T>
class MultiArrayTable final {

#define MEMBERS(x) \
	x(T, min) \
	x(T, max) \
	x(T*, offsets) \
	x(T*, rows)

DECLARE_PRIVATE(MEMBERS)
#undef MEMBERS
{{< / highlight >}}

Yes, macros.
C++ lacks compile-time reflection.
COAT uses macros to add a bit of meta data, making the list of members and their types available to other templates like `coat::Struct`.
This way we get access to each member from the generated code.

The heavy lifting in the _equi-join_ operator is done by `ht.iterate()`.
It is a custom function added to the wrapper object with CRTP.
Hence, the implementation details of the "hash table" can be encapsulated inside its header file, even for code generation.
As a user of the data structure, we do not have to know how to iterate over all entries with the same key.
We use the provided convenience function.
For details, check out the file [_include/MultiArrayTable.h_](https://github.com/tetzank/sigmod18contest/blob/master/include/MultiArrayTable.h#L56).

{{< todo >}}
- show how `ht.iterate()` is implemented
- explain `coat::Struct` and how to add `iterate()` to it
{{< / todo >}}

Just for fun, I implemented additional join variants:
joining with a [unique column][joinunique] using a simple [array table][AT], and a [semi-join][semijoin] using a [bitset][BT] to check if there is a join partner.
You can see it as a form of strength reduction.
We are replacing the generic join operator with cheaper versions if the data allows it.
A minor speedup is the result.
Follow the links and check out the code if you are curious.
[joinunique]: https://github.com/tetzank/sigmod18contest/blob/master/include/JoinUniqueOperator.h
[AT]: https://github.com/tetzank/sigmod18contest/blob/master/include/ArrayTable.h#
[semijoin]: https://github.com/tetzank/sigmod18contest/blob/master/include/SemiJoinOperator.h
[BT]: https://github.com/tetzank/sigmod18contest/blob/master/include/BitsetTable.h


The last operator in each pipeline is the _projection_ operator.

{{< highlight "C++" >}}
template<class Fn>
void codegen_impl(Fn &fn, CodegenContext<Fn> &ctx){
	// iterate over all projected columns
	for(size_t i=0; i<size; ++i){
		auto [column, relid] = projections[i];
		// fetch value from projected column
		auto val = loadValue(fn, *column, ctx.rowids[relid]);
		// add value to implicit sum on the projected column
		ctx.results[i] += val;
	}
	// count number of elements in sum
	++ctx.amount;
}
{{< / highlight >}}

For each projected column, we fetch the value and add it to the implicit sum on this projected column.
Additionally, we count the number of elements we have in the sum.
This counter is used to distinguish an empty result set from a sum which just happens to be zero.

To return the results back to the caller of the generated function, we must store the sums to memory.
After all operators in the pipeline generated their code, the following function is called to generate an epilogue.

{{< highlight "C++" >}}
template<class Fn>
void codegen_save_impl(Fn &fn, CodegenContext<Fn> &ctx){
	// get memory location to store result tuple to from an argument
	auto &projaddr = std::get<2>(ctx.arguments);
	// iterate over all projected columns
	for(size_t i=0; i<size; ++i){
		// store sum to memory
		projaddr[i] = ctx.results[i];
	}
}
{{< / highlight >}}

It stores each sum in contiguous memory.
The pointer was passed as an argument to the generated function.
The counter for the number of elements in the projection is passed as return value back to the caller.

And, that's it!
Code generation can be that easy.


# Experimental Evaluation

If you build the prototype and run the workload provided by the contest, you will see that the execution with the AsmJit backend is much faster than the LLVM backend.
Compilation latency is the main difference here.
The workload consists of a lot of short running queries which benefit from the very low compilation latency of AsmJit.
Even disabling optimizations in LLVM does not close the gap for the compilation latency, and the execution time suffers considerably from it.

Here are some measurements from my workstation.
Depending on your machine, you might get quite different numbers, but the trend should be similar.

Back End | Compilation Latency | Execution Time |
 ------- | -------------------:| --------------:|
AsmJit   |                5 ms |         550 ms |
LLVM -O0 |              341 ms |         768 ms |
LLVM -O1 |              650 ms |         513 ms |
LLVM -O2 |              663 ms |         521 ms |
LLVM -O3 |              660 ms |         567 ms |

In database research, the use of LLVM as JIT compiler seems to be the standard.
JIT assemblers are usually neglected for being too hard to use and producing inefficient code.
Furthermore, they are not portable.
Well, COAT makes them easy to use and code efficiency is not so bad as we see by the results.
Portability remains an issue.
I still like to have them on the table to give me a baseline, especially a lower bound for the compilation latency.
A JIT assembler shows me what is possible with a simple 1:1 mapping of C++ expressions to assembly instructions and how much a JIT compiler can improve on that given I invest time in optimizations.
An unoptimized build with LLVM does not provide me with the same lower bound.

These results should be taken with a grain of salt.
This is just a simple prototype running a single workload.
It needs to be shown if a JIT assembler like AsmJit can be used as a viable alternative to LLVM for short running queries in a more realistic setting.
COAT makes it easily accessible, so why not try it out.

{{< todo >}}
- plot with measurements of public workload, comparing the two backends
- breakdown of execution time
{{< / todo >}}


# Conclusion

I hope this tutorial was informative and not too long.
Code generation has a small learning curve to get into the right mindset.
We are generating code which is executed later.
We have to keep in mind what is run when.
COAT helps to concentrate on the control flow of the generated code by hiding the gritty details of the compiler APIs.
It streamlines the process to the point where code generation is not hard anymore.

In databases, more and more researchers and companies are adopting JIT compilation to squeeze more performance from the hardware.
Now is a good time to jump in on it.
