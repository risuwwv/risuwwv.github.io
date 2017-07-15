---
layout: post
title: Approximating square root
---

*In this post, I am going to present several square root approximations.*

All the code below can be found [in my github](https://github.com/risuwwv/ApproxSqrt) under BSD licence.

# Table of Contents
[Square root approximation](#sqrt)
<br>
[Constexpr](#Constexpr)
<br>
[Useful Functions](#Useful)
<br>
[Vectorization](#Vectorization)
<br>
[Details](#Details)
<br>
[Future work](#future)

<div id='sqrt'/>
## Square root approximation:

Some times ago, I asked myself how the square root function might be implemented and approximated. 
After a quick look on Wikipedia I went on and implemented the [Babylonian method](https://en.wikipedia.org/wiki/Methods_of_computing_square_roots):

  * Start from a rough estimate `y = 0.5f*(1.0f + x)`
  * Iterate the following: `y = 0.5f*(y + t/y)`

I will not give details on why it converges, but assuming it does, the limit verifies `2.0*y – y = t/y` which would mean `y^2 = t` if we worked with reals instead of `float`. 

However, the number of iterations needed can be quiet important for small and big numbers. 

Another minor drawback is that the convergence point is not always equal to `sqrtf(x)`. 
There may be a difference of +/-1 on the binary representations of `res` and `sqrtf(x)` because of rounding: 

For example for `x = 1.17549477e-38f` we have:

```cpp 
res = 1.08420243e-19f
0.5f*(res + x / res) = res
```

and

```cpp 
res2 = 1.08420230e-19f
0.5f*(res2 + x / res2) = res
```

but	
	
```cpp 
x-static_cast<double>(res)*res = -1.4012991325159946e-45
x-static_cast<double>(res2)*res2 = 1.4012982972770227e-45
```
so `res2` is nearer the correct value (which can't be represented exactly as float) even if `res` is the fixed point.

To take this into account, I added a final step that does this correction:

```cpp 
double tmp = static_cast<double>(res);

//when x - res*res == 0.0f, the direction to search is unclear if done with float instead
double diff = x - tmp*tmp; 
uint32_t bits = bit_cast<uint32_t>(res);
if (diff < 0)
{
   --bits;
}
else
{
   ++bits;
}

float res2 = bit_cast<float>(bits);
double tmp2 = static_cast<double>(res2);
double diff2 = x - tmp2*tmp2;

if (abs(diff2) < abs(diff))
{
   res = res2;
}
```

On my machine in the default rounding mode, this gives the exact same result as sqrtf but it is a lot slower!

One trick I found on [stackoverflow](http://stackoverflow.com/questions/19611198/finding-square-root-without-using-sqrt-function) (by user **mvp**) is to use [frexp](http://en.cppreference.com/w/cpp/numeric/math/frexp) and [ldexp](http://en.cppreference.com/w/cpp/numeric/math/ldexp) to move the computation to a smaller interval where it converges faster.

This gives a pretty good approximation for normalized floats and converges in at most three iterations of `y = 0.5f*(y + t/y)`.

However, the following technique found on [Wikipedia](https://en.wikipedia.org/wiki/Methods_of_computing_square_roots#Approximations_that_depend_on_the_floating_point_representation) gives faster implementations (between 1.4 to 2.8 faster):

```cpp 
uint32_t bits = (bit_cast<uint32_t>(x) >> 1) + (1 << 29) - (1 << 22);
res = bit_cast<float>(bits);
```

The bit trick above followed by 0-4 iterations of the Babylonian method gives the following:

```cpp 
float testSqrt1(float x) noexcept
{
   float res = 0.0f;

   uint32_t bits = (bit_cast<uint32_t>(x) >> 1) + (1 << 29) - (1 << 22);

   res = bit_cast<float>(bits);

   return res;
}

float testSqrt2(float x) noexcept
{
   float res = 0.0f;

   uint32_t bits = (bit_cast<uint32_t>(x) >> 1) + (1 << 29) - (1 << 22);

   res = bit_cast<float>(bits);
   res = (res + x / res) * 0.5f;

   return res;
}

float testSqrt3(float x) noexcept
{
   ...
   
   res = (res + x / res) * 0.5f;
   res = (res + x / res) * 0.5f;

   return res;
}
```

Where testSqrt4 an testSqrt5 do respectively 3 and 4 steps of `res = (res + x / res) * 0.5f`.

After some tests, the following gives better results:

```cpp 
uint32_t bits = (bit_cast<uint32_t>(x) >> 1) + 532362861;
```

Where `532362861` is `(1 << 29) - (1 << 22) - 313747` and was found by searching values in the neighbourhood of `(1 << 29) - (1 << 22)`: it is the value that minimize the maximal relative error on all normalized numbers.

This leads to the following corresponding functions:

```cpp 
float testSqrt6(float x) noexcept
{
   float res = 0.0f;
	
   uint32_t bits = (bit_cast<uint32_t>(x) >> 1) + 532362861;

   res = bit_cast<float>(bits);

   return res;
}

float testSqrt7(float x) noexcept {...}
float testSqrt8(float x) noexcept {...}
float testSqrt9(float x) noexcept {...}
float testSqrt10(float x) noexcept {...}
```

that are similar to testSqrt1, ..., testSqrt10 but with the modified constant.

The a simplified version of the micro benchmark I ran was:

```cpp 
float sum = 0;
for (uint32_t i = 8388608; i < 2139095040; ++i)
{
   float val = bit_cast<float>(i);
   float res = testSqrt8(val);
   sum += res;
}
	
cout << "sum:" << sum << endl;
```

It computes the sum of the square roots of all the normalized float. 

This is done because compilers are able to optimize away loops without side effects so you can't just time the following:

```cpp 
for (uint32_t i = 8388608; i < 2139095040; ++i)
{
   float val = bit_cast<float>(i);
   float res = testSqrt8(val);
}
```

In general it is a good idea to check the assembly code when doing micro-benchmarks as compilers are quiet smart.
The exact code I used is available in the [repository](https://github.com/risuwwv/ApproxSqrt/src).

With clang, I got the following timings:

| Function | time (s)|  speedup |
| $$sqrtf$$    |    8.93 |       1  |
|$$\color{red}{testSqrt1}$$ |    2.62 | **3.40** |
|$$\color{red}{testSqrt2}$$ |    3.70 | **2.42** |
|$$testSqrt3$$ |    7.47 |     1.20 |
|$$testSqrt4$$ |    11.15 |     0.80 |
|$$testSqrt5$$ |    14.82 |     0.60 |
|$$\color{red}{testSqrt6}$$ |    2.66 | **3.35** |
|$$\color{red}{testSqrt7}$$ |    3.67 | **2.43** |
|$$testSqrt8$$|    7.46 |     1.20 |
|$$testSqrt9$$ |    11.14 |     0.80 |
|$$testSqrt10$$|    14.79 |     0.60 |

As you can see, having three iterations of more is too expensive compared to regular sqrtf, while having lower precision as described below.

The following table shows the maximal relative error of the different algorithms on normalized numbers.
Given this error bound, the algorithms can be extended to bigger intervals than normalized float, the lower bounds (as bit_cast<uint32_t>) are also showed below:

| Function                  | Lower bound | Max relative error|
| $$sqrtf$$                 |   **0**     |  1.19209e-07      |
|$$\color{red}{testSqrt1}$$ |  5999996    |  0.206136f        |
|$$\color{red}{testSqrt2}$$ |   6300003   |  0.00623746f      |
|$$testSqrt3$$              |  6497369    |  6e-06f           |
|$$testSqrt4$$              |    3962353  |  2.63158e-07f     |
|$$testSqrt5$$              |   1238799   | 2.68155e-07f      |
|$$\color{red}{testSqrt6}$$ |    6872893  |  0.0695972f       |
|$$\color{red}{testSqrt7}$$ |   6813933   |  0.00130169f      |
|$$testSqrt8$$              |    6778363  |  5.95071e-07f     |
|$$testSqrt9$$              |  4023293    |  1.78727e-07f     |
|$$testSqrt10$$             |  2100551    | 1.78727e-07f      |

Relative errors become absurdly big on denormalized for all 10 versions (testSqrt10 has 17523.9 for example...), however absolute error stays pretty low.

On denormalized and positive zero:

| Function                  | Max absolute error |
|$$\color{red}{testSqrt1}$$ | 6.61216e-39        |
|$$\color{red}{testSqrt2}$$ | 1.65304e-39        |
|$$testSqrt3$$              | 4.1326e-40         |
|$$testSqrt4$$              | 1.03315e-40        |
|$$testSqrt5$$              | 2.58287e-41        |
|$$\color{red}{testSqrt6}$$ | 6.28653e-39        |
|$$\color{red}{testSqrt7}$$ | 1.57163e-39        |
|$$testSqrt8$$              | 3.92907e-40        |
|$$testSqrt9$$              | 9.82268e-41        |
|$$testSqrt10$$             | 2.45564e-41        |

<div id='Constexpr'/>  
### Constexpr:

I thought about constexprness, but it seams that bit_cast is a problem: 
The only way to access bits in a float at compilation time at the moment seems to be [this](http://brnz.org/hbr/?p=1518), 
which would not be good if it was called on runtime parameter, so we would need two versions of each.

<div id='Useful'/>  
### Useful functions for float:

[std::isfinite](http://en.cppreference.com/w/cpp/numeric/math/isfinite)/[std::isnormal](http://en.cppreference.com/w/cpp/numeric/math/isnormal)/[std::isinf](http://en.cppreference.com/w/cpp/numeric/math/isinf)/[std::isnan](http://en.cppreference.com/w/cpp/numeric/math/isnan)/[std::fpclassify](http://en.cppreference.com/w/cpp/numeric/math/fpclassify)
<br>
[std::logb](http://en.cppreference.com/w/cpp/numeric/math/logb)/[std::ilogb](http://en.cppreference.com/w/cpp/numeric/math/ilogb)
<br>
[std::scalbn](http://en.cppreference.com/w/cpp/numeric/math/scalbn)
<br>
[std::modf](http://en.cppreference.com/w/cpp/numeric/math/modf)
<br>
[std::nearbyint](http://en.cppreference.com/w/cpp/numeric/math/nearbyint)/[std::rint](http://en.cppreference.com/w/cpp/numeric/math/rint)
<br>
[std::fesetround](http://en.cppreference.com/w/cpp/numeric/fenv/feround)/[std::fegetround](http://en.cppreference.com/w/cpp/numeric/fenv/feround)
<br>
[std::nexafter](http://en.cppreference.com/w/cpp/numeric/math/nextafter)/[nexttoward](http://en.cppreference.com/w/cpp/numeric/math/nextafter)
<br>
[std::trunc](http://en.cppreference.com/w/cpp/numeric/math/trunc)/[std::floor](http://en.cppreference.com/w/cpp/numeric/math/floor)/[std::ceil](http://en.cppreference.com/w/cpp/numeric/math/ceil)/[std::round](http://en.cppreference.com/w/cpp/numeric/math/round)

<div id='Vectorization'/>  
### Vectorization:

Another thing we can do is to take advantage of the support for [vectorized](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions) operations in modern CPUs.

As my own machine supports [AVX2](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions), it can process `8 floats/int32_t` or `4 doubles/int64_t` in a single instruction.

I will not get into details here as it deserves its own post but vectorized versions can be build easily from the branch-less implementations above.

As using intrinsics ($$\color{red}{warning}$$: throughputs are inverted in the [IntrinsicsGuide](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#techs=AVX,AVX2&text=abs&expand=695,4635,696,112,3775,5220,305,575): the lower the better) can become ugly pretty fast, a wrapper library can help.
[libsimdpp](https://github.com/p12tic/libsimdpp) and [boost.simd](https://github.com/NumScale/boost.simd) are two libraries using [expression template](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Expression-template) to allow vectorization while keeping readability and maintainability.

Vectorized versions of testSqrt6 to 10 are in my repository but I will talk more about this in the next post(s).
 
Speed-ups are measured compared with non vectorized versions:
 
|       Function             | speed-up |
| $$\_mm256\_sqrt\_ps$$      |7.4       |	
| $$\color{red}{testSqrt6}$$ |5.8       |
| $$testSqrt7$$ 			 |4.0       |
| $$testSqrt8$$		         |4.0       |
| $$testSqrt9$$		         |4.0       |
| $$testSqrt10$$	         |4.0       |

I expected the speed-up to be near 8, but according to the intrinsic guide _mm256_div_ps has 21 cycles latency while _mm_div_ps has only 13. 

I found in [this](http://www.intel.com/content/www/us/en/architecture-and-technology/64-ia-32-architectures-optimization-manual.html), page 422, that the 256-bit VDIVPS instruction executes with 128-bit data path before Skylake...

If somebody has an explanation for why they did not made _mm256_div_ps in 13 cycles too, I would be glad to know (too many transistors compare to expected usefulness?)

<div id='Details'/>  
### Details:

All the measures were done on an Intel i7-4790K running at 4.32GHz.
I built in 64bits mode with MSVC 19.00.23918, Clang 3.7.0 and gcc 5.1.0.

Clang command line is: 

`clang++ -o bench.exe bench.cpp approxSqrt.cpp -std=c++14 -Weverything -Werror -Wno-c++98-compat -Wno-used-but-marked-unused -Wno-padded -march=native -O3 -DNDEBUG`

As gcc does not have -Weverything, the g++ command line is pretty long, so you can find it in the repository.


<div id='future'/>
## Future work

Ameliorations and extensions of this include:

  * Making versions that work better for denormalized.
  * Versions that always overestimates/underestimates can be useful in some cases.
  * It is also possible to write versions taking and returning integer.
  * Similarly support of the type double could be studied.
  * Measures could be made on a practical case instead of a micro-benchmark.

If you have written or read interesting articles on function approximations I would be interested to know about it.

Please don't hesitate to post feedback or correct me if any of the above is inexact!
