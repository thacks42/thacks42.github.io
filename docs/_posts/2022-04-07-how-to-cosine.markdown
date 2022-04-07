---
layout: post
title:  "How (not) to cosine. Part 1"
---

## To cosine or not to cosine

There was a recent reddit post about implementing `cos(x)` from scratch in C.
<https://www.reddit.com/r/programming/comments/tr8zrq/implementing_cosine_in_c_from_scratch/>

In which the author explored two different approaches to implementing (an approximate) cosine function,
which can serve as a great example to show why "textbook math"[^textbookmath] approaches to numerical algorithms can often fall short (or alternatively: why the whole field of numerics exists).

This post is not intended to poke fun at the author, but rather to show some numerical techniques that are usually not taught in normal algebra or calculus classes.

### Why Mr. Taylor isn't your friend

When faced with the problem of implementing a function to calculate `cos(x)` a natural idea might be to use a truncated Taylor series[^taylor].

In fact the Taylor series for `cos(x)` follows a rather nice pattern:

`cos(x) = 1 - x^2/2! + x^4/4! - x^6/6! + ...`

And while Taylor series are a great tool for ~~making students lose their will to live~~ proofs in calculus classes,
using a truncated Taylor series to approximate an analytic function will rarely yield great results.

To understand why, we have to take a closer look at how polynomial approximations work.
First of all even though your original Taylor series might "converge everywhere", your truncated Taylor series will actually have a finite interval on which it approximates the desired function well.
Outside of that interval it will quickly diverge to positive or negative infinity.

![Error of Taylor approximations](/assets/how_to_cosine/error_taylor_order_6.png)

One striking feature in the error plot is the monotonicity: the further away from `x = 0` we get, the larger the error. This might be visually pleasing but other than that, it is actually awful, because...

### Polynomials want to be curvy

For a given interval we care about the **maximum** error of our approximation. To improve this, we can obviously just increase the order of the polynomial i.e. use more terms of the Taylor approximation,
but this means that our function takes longer and longer to compute, so if we have decided on some number of terms that we deem "sufficiently few to be fast enough to compute",
we want to minimize the polynomials maximum error[^actual_function_design].

And here we encounter an interesting property of polynomials: They want to be curvy! The error of our Taylor polynomial grows much faster than necessary due to the fact that its error is not "allowed to oscillate".
In general truncated Taylor polynomials are "well behaved" close to **one point**. You can choose this point however you want, but you can only choose a single point. Not an interval, which is what we actually care about.

So how do we design a polynomial that minimizes the maximum error over an **interval**? We evaluate our function at `n` points in the interval and then fit a polynomial of degree `n-1` through those points.
Here we can choose whichever points we want (e.g. we could just space out our `n` points evenly across the interval, group them all in the center etc.).
As it turns out, choosing the so called "Chebyshev nodes"[^cheby_nodes] over our interval will often yield near optimal results.

![Taylor vs Chebyshev error](/assets/how_to_cosine/error_taylor_cheby_comparison.png)

Now that's a huge difference! Somewhere at around `x = 2.5` the error of the Taylor approximation "explodes" whereas the Chebyshev approximation follows the original function over the whole interval,
even though it has the same order!

If we zoom in, we discover something interesting....

![Taylor vs Chebyshev error](/assets/how_to_cosine/error_taylor_cheby_comparison_zoomed.png)

The error of the Chebyshev approximation oscillates over the whole interval. By "allowing" the polynomial to be "more curvy", we got a much better maximum error overall.

### Range reduction (how much cosine do you need anyway?)

When designing a numerical approximation, we should always try to make the interval over which the approximation has to be correct as small as possible.
For `cos(x)` there are a few obvious way in which this can be done.

First of all we know that `cos(x) = cos(-x)`, which lets us disregard all negative values if we just compute the absolute value of `x`:
```cpp
float my_cos(float x){
    x = abs(x);         // cos(-x) == cos(x)
    float result = .... // magic
    return result;
}
```

Next we know that `cos(x)` is a periodic function that repeats at every multiple of 2π. Which in turn means that if we are able to find the largest multiple of 2π less than `x`,
we can subtract that from `x` and calculate `cos` of the result:


```cpp
float my_cos(float x){
    x = abs(x);         //      cos(-x) == cos(x)
    x = fmod(x, 2*pi);  // cos(x + 2pi) == cos(x)
    float result = .... // magic
    return result;
}
```

But that's not all! If our goal is to restrict the interval over which our approximation has to be good, we can actually do a bit more:

![cosine quadrants](/assets/how_to_cosine/cosine_quadrants.png)

The first quadrant is mirrored and repeated for three more times. So our approximation can be reduced to the interval `[0, pi/2]` if we do some clever mirroring:

```cpp
float my_cos(float x){
    x = abs(x);         //      cos(-x) == cos(x)
    x = fmod(x, 2*pi);  // cos(x + 2pi) == cos(x)
    
    bool result_negative;
    if(x > pi){
        if(x > 3*pi/2){                 // quadrant 3
            result_negative = false;
            x = 3*pi/2 - x;             // mirror vertically
        }
        else{                           // quadrant 2
            result_negative = true;
            x = x - pi;                 // shift to the left
        }
    }
    else{
        if(x > pi/2){                   // quadrant 1
            result_negative = true;
            x = pi - x;                 // mirror vertically
        }
        else{                           // quadrant 0
            result_negative = false;
                                        // no shifting/mirroring needed
        }
    }
    
    float result = .... //magic
    if(result_negative) result = -result; // restore the sign
    return result;
}
```

Reducing our interval of interest to `[0, pi/2]` should improve our error. We will also include a fifth order Chebyshev approximation for comparison:

![Error of Taylor approximations](/assets/how_to_cosine/error_taylor_cheby_comparison_zoomed_pi_over_two.png)


That's another serious improvement. The maximum error of our sixth order Chebyshev approximation is now well below `10^-5` in fact we got it down to `4.3*10^-7`(!)

And the error of the fifth order Chebyshev approximation manages an error less than `7.8*10^-6`,

whereas the maximum error of the Taylor polynomial sits firmly at `8.9*10^-4` 

---
[^textbookmath]: "textbook" here refers to typical calculus textbooks. Numerics textbooks obviously do exist and teach the proper techniques, but they often aren't part of a typical university math course.
[^taylor]: [Taylor series on wikipedia](https://en.wikipedia.org/wiki/Taylor_series)
[^actual_function_design]: In real world applications this is often done the other way around: we decide on a maximum error bound and then find the lowest order polynomial that achieves said goal.
[^cheby_nodes]: [Chebyshev Nodes on wikipedia](https://en.wikipedia.org/wiki/Chebyshev_nodes)




