---
title: "X-Macros"
date: 2019-04-23T16:34:00+02:00
---

A lot of C++ developers try to avoid preprocessor macros like the plague.
There are genuine reasons for that.
Macros might look like functions, but they behave differently, resulting in confusing bugs when not treated carefully.
But even in modern C++, macros still have their use cases.
In this post, I want to talk about a special kind of macro called X-macro which is mostly used to generate various code fragments from a single list of elements.


# A Macro is not a Function

When I started developing in C/C++, I used macros as constants and small inlined functions.
This works for simple expressions, if one writes the macro carefully, but not for code fragments consisting of multiple statements.
Here is a simple expression macro squaring a passed value:

```
#define SQUARE(value) value * value
```

The macro is expanded to the passed value multiplied by itself.
Looks easy enough.
What could possibly go wrong?
Well, a lot actually.
What is the result of `SQUARE(i+1)`?

In a function call, the expression `i+1` is evaluated at the call side and the resulting value passed as parameter to the function.
Call-side evaluation of parameters is not happening for macros.[^1]
[^1]: Parameters are macro-expanded at call side, see [Argument Prescan](https://gcc.gnu.org/onlinedocs/cpp/Argument-Prescan.html#Argument-Prescan).
The whole parameter expression is inserted everywhere the parameter is used in the macro definition, resulting in `i+1 * i+1` which is not what we intended to calculate.
Operator precedence gets in our way.
The fix is obvious: put everything in parentheses!

```
#define SQUARE(expr) ((expr) * (expr))
```

The reason for this behavior is easy to understand: the preprocessor does not know anything about the semantics of the expressions.
Basically, it just copies text around following its expansion rules, not knowing what the text means and if the assembled code makes any sense for the following compilation step.
This separation of concerns keeps tools simple, but it can lead to unexpected pitfalls.
If you feel curious about other common problems, you can check out the ['Macro Pitfalls'](https://gcc.gnu.org/onlinedocs/cpp/Macro-Pitfalls.html#Macro-Pitfalls) section in the documentation.

I do not use macros as inlined functions anymore.
A normal function has way less surprises in store than a macro.
The compiler is usually also better at deciding when to inline a function call.
If need be, one can still force the compiler to inline the function call with the function attribute `[[gnu::always_inline]]`, or a similar attribute for your compiler of choice.



# Nested Expansions

After a macro is expanded, the whole resulting body of the macro is scanned another time for any known macros to be expanded.
This expansion rule allows us to write nested macros, leading to so called X-macros.

The common "Hello, World!"-example for X-macros is an enumeration and an array holding a string representation for each element of the enum.

{{< highlight "C++" >}}
struct colors {
	enum enumeration {
		black, white, red, blue, green
	};
	static constexpr std::array<const char*,5> names {
		"black", "white", "red", "blue", "green"
	};
};
{{< / highlight >}}

Updates to the enum requires changes to the array as well.
It would be nice to have just one list of elements and generate the enum and array from that list.
X-macros can help us here.

{{< highlight "C++" >}}
struct colors {
	// list of elements
	#define COLORS \
		X(black)   \
		X(white)   \
		X(red)     \
		X(blue)    \
		X(green)

	// put each element into the enum
	enum enumeration {
		#define X(color) color,
		COLORS
		_number_of_elements
		#undef X
	};

	// string for each enum element
	static constexpr std::array<const char*,_number_of_elements> names {
		#define X(color) #color,
		COLORS
		#undef X
	};
};
{{< / highlight >}}

First, we define the macro `COLORS` containing the list of elements.
Each element is enclosed by another macro `X()`, yet to be defined.
Next, we define the enumeration.
Inside the body of the enum, we define the macro `X()` to echo the parameter it gets followed by a comma.
On the next line, the macro `COLORS` is called which will use the newly defined macro `X()` inside of its own expansion.
The result of the nested expansion is a list of all elements separated by commas.

Additionally, we add the pseudo-element `_number_of_elements` at the end of the enumeration to automatically count the number of elements in the enum.
It is used as template parameter for `std::array`. If we would use C++17 class template argument deduction, we could omit the template parameter for `std::array` and would not need `_number_of_elements`.

At the end, we explicitly undefine the macro `X()`, so that we can define it in a different way for the string array.
The initializer list for the array is generated similarly to the content of the enum.
We define the macro `X()` before expanding `COLORS`.
This time, we want to generate a string representation for each element.
We convert the macro parameter to a string literal with the preprocessing operator `#` in front of the parameter, see [Stringizing](https://gcc.gnu.org/onlinedocs/cpp/Stringizing.html#Stringizing).
It will generate strings in the same order as the enum.


# More Nesting

Undefining and redefining the macro `X()` is a bit clunky.
Ideally, we only want the list of elements and one macro to generate all the code for us.
Something like this:

{{< highlight "C++" >}}
struct colors {
// list of elements
#define COLORS(X) \
	X(black)      \
	X(white)      \
	X(red)        \
	X(blue)       \
	X(green)

DECLARE_ENUM(COLORS)
};
{{< / highlight >}}

Notice that the list of elements `COLORS` now takes the macro `X()` as a parameter.
This way, we do not have to undefine and redefine `X()`.
We just pass a different macro to `COLORS`.
The parameter is inserted into the macro body during expansion, replacing X, and then the nested expansion is triggered.
The macro `DECLARE_ENUM()` hides most of the details.
The definition is the following:

{{< highlight "C++" >}}
#define  ENUM_MEMBER(element) element,
#define ARRAY_MEMBER(element) #element,

#define DECLARE_ENUM(ELEMENTS)                                                   \
	enum enumeration {                                                           \
		ELEMENTS(ENUM_MEMBER)                                                    \
		_number_of_elements                                                      \
	};                                                                           \
	static constexpr std::array<const char*,_number_of_elements> names { \
		ELEMENTS(ARRAY_MEMBER)                                                   \
	};
{{< / highlight >}}

During the expansion of `DECLARE_ENUM()` two nested expansion are triggered one after the other.
First, the list of elements `COLORS` is inserted into the enum and the array, replacing ELEMENTS.

{{< highlight "C++" >}}
	enum enumeration {
		COLORS(ENUM_MEMBER)
		_number_of_elements
	};
	static constexpr std::array<const char*,_number_of_elements> names {
		COLORS(ARRAY_MEMBER)
	};
{{< / highlight >}}

Next, the inserted `COLORS` is expanded in the enum, using `ENUM_MEMBER()` as the definition of X, and in the initializer list of the array, using `ARRAY_MEMBER()`.

{{< highlight "C++" >}}
	enum enumeration {
		ENUM_MEMBER(black)
		ENUM_MEMBER(white)
		ENUM_MEMBER(red)
		ENUM_MEMBER(blue)
		ENUM_MEMBER(green)
		_number_of_elements
	};
	static constexpr std::array<const char*,_number_of_elements> names {
		ARRAY_MEMBER(black)
		ARRAY_MEMBER(white)
		ARRAY_MEMBER(red)
		ARRAY_MEMBER(blue)
		ARRAY_MEMBER(green)
	};
{{< / highlight >}}

Finally, `ENUM_MEMBER()` and `ARRAY_MEMBER()` are expanded on each element of the list.

{{< highlight "C++" >}}
	enum enumeration {
		black,
		white,
		red,
		blue,
		green,
		_number_of_elements
	};
	static constexpr std::array<const char*,_number_of_elements> names {
		"black",
		"white",
		"red",
		"blue",
		"green",
	};
{{< / highlight >}}


# Conclusion

Defining the list of elements in a macro enclosed by yet another macro still feels a bit clunky.
The benefit we get from all this is a single definition of the elements which makes it much more update friendly.
No more forgotten updates!

I hope you do not get the impression that X-macros are limited to enums.
I recently used X-macros to add a bit of metadata to classes and structures in such a way that I get a very simple compile-time reflection of their members.
My list of elements was the members with their types.

See also wikibooks for [more examples](https://en.wikibooks.org/wiki/C_Programming/Preprocessor#X-Macros).
