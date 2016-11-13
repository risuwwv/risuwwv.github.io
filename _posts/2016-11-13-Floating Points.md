---
layout: post
title: About float and double.
---

*For this first post, I planned to present several square root approximations, but since I manipulate the bits of the float representation, I need to first write about float/double in C++.*


All the code below can be found [in my github](https://github.com/risuwwv/ApproxSqrt) under BSD licence.

# Table of Contents
[Floating points](#Floatingpointnumbers)
[float](#float)
[double](#double)
[bit_cast](#bit_cast)
[Limitations](#Limitations)
[Comparing floats](#Comparingfloats)
[Fun facts](#Funfacts)
[Links](Links)

<div id='Floatingpointnumbers'/>
# Floating point numbers:

The IEC 559/IEEE 754 is a [technical standard for floating-point computation](https://en.wikipedia.org/wiki/IEEE_floating_point).
In C++, compliance with IEC 559 can be checked with the [is_iec559](http://en.cppreference.com/w/cpp/types/numeric_limits/is_iec559) member of `std::numeric_limits`. 
Nearly all modern CPUs from Intel, AMD and ARMs and GPUs from NVIDIA and AMD should be compliant.

float and double computations can be done with different [rounding modes](http://www.cplusplus.com/reference/cfenv/fegetround/)
All the tests in this article were done with the default `Round to nearest` mode.

<div id='float'/>
# float:
The type `float` is 32 bits: 1 **$$\color{red}{sign}$$** bit, 8 bits of **$$\color{blue}{exponent}$$** and 23 bits of **$$\color{green}{fraction}$$**.
 
 
### When the exponent is 255:

The float is either infinity or NaN (**N**ot **a** **N**umber):

  * Negative infinity is $$0b\color{red}1\color{blue}{11111111}\color{green}{0...0}$$
  * Positive infinity is $$0b\color{red}0\color{blue}{11111111}\color{green}{0...0}$$
  * NaNs may be **signalling** or **quiet**.
  
Comparisons with NaNs (>, >=, <, <= and ==) always return false even when both sides have the same bit pattern:

`NaN != NaN`
 
So if you want to accept only non negative numbers and NaNs (**including ones with 1 as sign bit**), you may change

```cpp 
assert(x>= 0); 
```

into

```cpp 
assert(!(x<0)); 
``` 
 
### When the exponent is in $$[1, 254]$$:

The bits are interpreted as $$(-1)^s\cdot2^{e-127}\cdot(1.f)$$.


### When exponent is $$0$$:

The bits are interpreted as $$(-1)^s\cdot2^{-126}\cdot(0.f)$$.

Because of this format, there are **two zeros** that compare equal but have different bits:

-0.0f : $$0b\color{red}1\color{blue}{00000000}\color{green}{0...0}$$
+0.0f : $$0b\color{red}0\color{blue}{00000000}\color{green}{0...0}$$

Other numbers with exponent $$0$$ are called **denormalized**.

The smallest positive float is 1.401e-45#DEN: $$0b\color{red}0\color{blue}{00000000}\color{green}{0...01}$$. 
The biggest finite float is 3.40282347e+38: $$0b\color{red}0\color{blue}{11111110}\color{green}{1...1}$$.

<div id='double'/>
# double:
The type `double` is 64 bits: 1 **$$\color{red}{sign}$$** bit, 11 bits of **$$\color{blue}{exponent}$$** and 52 bits of **$$\color{green}{fraction}$$**. 
It follows the same rules but the offset on the exponent is 1023 instead of 127, and the special exponents are 0 and 2047.

<div id='bit_cast'/>
# bit_cast:
To access the bits of a float, the safest way is to use memcpy, as both reinterpret_cast and unions are conflicting with strong aliasing rules according to the [standard](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3337.pdf) (see 3.9 and 3.10-10):

```cpp 
uint32_t bits;
memcpy(&bits, &someFloat, sizeof(someFloat));
```

It can be wrapped in a template, which I chose to call bit_cast:

```cpp 
template<typename DstType, typename SrcType>
DstType bit_cast(SrcType src)
{
	static_assert(sizeof(DstType) == sizeof(SrcType), 
	"sizeof(DstType) != sizeof(SrcType)");

	DstType dst;
	memcpy(&dst, &src, sizeof(SrcType));
	return dst;
}
```

While looking for a better name for it, I found that chromium uses the [same one](https://chromium.googlesource.com/chromium/src/+/1587f7d/base/macros.h#76) so I gave up :D.

<div id='Limitations'/>
# Limitations:
 
Since float has a finite number of bits, it is obviously not able to represent all the real numbers in [1.401e-45, 3.40282347e+38].

This has some consequences: 

Dividing by 2.0f or multiplying by 0.5 is the same.
Dividing by 3.0f and multiplying by 1.0f/3.0f is not the same:

`0.00472378824f * (1.0f/3.0f) = 0.001574596`**`16`**
`0.00472378824f / 3.0f = 0.001574596`**`04`**

Because 1/3 can’t be represented exactly as a float:

`1.0f/3.0f == 0.3333333**4**3f with round to nearest`

Another consequence of this is that the following loop is infinite:  

```cpp 
for(float f = 0.0f; f < 20000000.0f; ++f){} 
```

as `16777216.0f + 1.0f` equals `16777216.0f`: ++f will never reach 20000000.0f !

  
Since the mantissa is only 24 bits (including the implicit leading 1 or 0), adding a number with a very small exponent to one with a very big exponent will leave the mantissa of the big number unaffected by the addition of the small one. 

This problem is particularly important when subtracting nearly equal numbers (or adding nearly opposite ones). 

Let’s say that I have two real numbers num1 and num2 both needing 26 bits of mantissa, and differing only on the least significant one.

Because float has only 24 bits of mantissa, the last two bits are already lost in previous computations at the point where num1 – num2 is computed, which leads to [**catastrophic cancellation**](https://en.wikipedia.org/wiki/Loss_of_significance).

Example:

`num1 = 16777216.0f + 1.0f`
`num2 = 16777216.0f`
`num1 - num2 = 0.0f`

But with double:

`num1 = 16777216.0 + 1.0`
`num2 = 16777216.0`
`num1 - num2 = 1.0`

Note that double has the exact same problem, but you need even bigger differences, like 10000000000000000.0+1.0.

They are different ways to overcome this when high precision is needed, like [interval arithmetic](https://en.wikipedia.org/wiki/Interval_arithmetic) or [this](http://www.cs.berkeley.edu/~jrs/papers/robustr.pdf).

<div id='Comparingfloats'/>
# Comparing floats:
In general, you should avoid direct equality comparisons for float, as they usually don't do what you may expect: 

Unless they are produced by the exact same steps, rounding errors may make two numbers you expect to be equal a bit different.

  * If you want two floats to have the exact same bits, do `bit_cast<uint32_t>(lhs) == bit_cast<uint32_t>(rhs)` (this considers +0 and -0 as different, but NaNs with same bits as equal).

  * If you want to check if they are approximately equal, do `fabs(lhs-rhs) <= epsilon*((fabs(lhs) < fabs(rhs) ? fabs(rhs) : fabs(lhs)` (from [The art of computer programming](http://www.amazon.com/gp/product/0321751043/)) with epsilon chosen for your application.

  * If you specifically want exactly the same bits with the exception of zeroes and NaNs, use the regular `==` operator.

<div id='Funfacts'/>
# Fun facts:
 * More than half of positive floats are inferior to 2 (exponents $$0$$ to $$127$$).
 
 * It is possible to use [radixsort for float](http://stereopsis.com/radix.html)!
 
 * Another idea that I found important is that as `float` are only 32bits, it is possible to test every value when trying to approximate/implement a function with a single parameter (sqrt takes less than a few minutes for example) if it is not too complex.

<div id='Links'/>
# Links:
[What Every Computer Scientist Should Know About Floating-Point Arithmetic](https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html)
[Wikipedia: Single-precision floating-point format](https://en.wikipedia.org/wiki/Single-precision_floating-point_format)
[Wikipedia: Double-precision floating-point format](https://en.wikipedia.org/wiki/Double-precision_floating-point_format)
[Kahan Summation](https://en.wikipedia.org/wiki/Kahan_summation_algorithm)

As this is my first post using Jekyll, please feel free to criticize both the form and the content!
