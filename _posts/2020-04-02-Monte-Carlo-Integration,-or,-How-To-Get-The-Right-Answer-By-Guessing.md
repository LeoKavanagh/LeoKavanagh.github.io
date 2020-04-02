... kind of.

Given a continuous function in one variable and lower and upper bounds,
we can calculate the area under the curve with Monte Carlo Simulation.

## Details 

Start with the defintion of the (arithmetic) mean of a function over a given interval:

![\bar{f(x)}=\frac{1}{N}\sum_{i=1}^N{f(x_{i})};a\leq{x}\leq{b}](https://render.githubusercontent.com/render/math?math=%5Cbar%7Bf(x)%7D%3D%5Cfrac%7B1%7D%7BN%7D%5Csum_%7Bi%3D1%7D%5EN%7Bf(x_%7Bi%7D)%7D%3Ba%5Cleq%7Bx%7D%5Cleq%7Bb%7D)

Now define the same thing in the continuous case rather than the discrete:

![\bar{f(x)}=\frac{1}{b - a}\int\limits_{a}^b{f(x)}dx](https://render.githubusercontent.com/render/math?math=%5Cbar%7Bf(x)%7D%3D%5Cfrac%7B1%7D%7Bb%20-%20a%7D%5Cint%5Climits_%7Ba%7D%5Eb%7Bf(x)%7Ddx)

Now, we can rearrange this expression to get a definition of the integral in terms of the mean and the width of the interval:

![\int_{a}^b{f(x)}dx=(b - a) * \bar{f(x)}](https://render.githubusercontent.com/render/math?math=%5Cint_%7Ba%7D%5Eb%7Bf(x)%7Ddx%3D(b%20-%20a)%20*%20%5Cbar%7Bf(x)%7D)

So, this implies that we can calculate the integral of
![f(x)](https://render.githubusercontent.com/render/math?math=f(x))
over the interval
![f(x)](https://render.githubusercontent.com/render/math?math=[a,b])
by first getting the average of the function over the interval
![f(x)](https://render.githubusercontent.com/render/math?math=f(x))
, and then multiplying by
![f(x)](https://render.githubusercontent.com/render/math?math=[a,b])
.

This is great, because very often we find ourselves in the situation
where evaluating
![f(x)](https://render.githubusercontent.com/render/math?math=f(x))
is easy for any given
![f(x)](https://render.githubusercontent.com/render/math?math=x)
, but _integrating_
![f(x)](https://render.githubusercontent.com/render/math?math=f(x))
over any given integral is very difficult indeed.

The simplest, most naive, approach is to Uniformly sample from the interval ![(b - a)](https://render.githubusercontent.com/render/math?math=(b%20-%20a)), and then evaluate ![f(x)](https://render.githubusercontent.com/render/math?math=f(x)) for each 
![x \sim Unif(a,b)](https://render.githubusercontent.com/render/math?math=x%20%5Csim%20Unif(a%2Cb)).

Now we have a sequence

<img src="https://render.githubusercontent.com/render/math?math=f(x_{1}), f(x_{2}), ..., f(x_{N});&space;x\sim&space;Unif(a,b)">

We can take the mean of this sequence,
multiply by the width of the interval,
and we get our discrete approximation of the integral:

![\int_{a}^{b} f(x) dx\approx(b - a) * \bar{f(x)};\  x\sim Unif(a,b)](https://render.githubusercontent.com/render/math?math=%5Cint_%7Ba%7D%5E%7Bb%7D%20f(x)%20dx%5Capprox(b%20-%20a)%20*%20%5Cbar%7Bf(x)%7D%3B%5C%20%20x%5Csim%20Unif(a%2Cb))

## Making This Less Mathematically Naive

Ultimately, the Monte Carlo approach leads to things like
[Markov Chain Monte Carlo](
https://en.wikipedia.org/wiki/Markov_chain_Monte_Carlo), via
things like [Importance Sampling](https://en.wikipedia.org/wiki/Markov_chain_Monte_Carlo).

Alternatively, there are sophisticated non-stochastic numerical
integration methods like
[Simpson's Rule](https://en.wikipedia.org/wiki/Simpson%27s_rule)
or
[Gaussian Quadrature](https://en.wikipedia.org/wiki/Simpson%27s_rule).

## Show Me The Code

I coded up a really simple, non-interactive version of this
in Scala using Breeze, Scala's main numerical methods library.

The function `mc_integrate` takes a function, number of Monte Carlo samples to generate,
and lower and upper bounds as arguments.

It evaluates the function at each randomly generated point point,
and then returns the approximation of the area under the curve.
That is, the approximation of the integral of the function
from the lower bound to the upper bound.

```
import breeze.linalg._
import breeze.stats.distributions.Uniform


def mc_integrate(fn: Double => Double, nsamp: Int, lower: Double, upper: Double): Double = {

    // Generate nsamp number of Uniform RVs over the interval [lower, upper]
    val x = Uniform(lower, upper).sample(nsamp)

    // evaluate the function at each randomly generated x value
    val f_x = x.map(fn)

	// mean of the function over the interval
    val mean_value = sum(f_x) / f_x.length

	// Return mean value of the function, multiplied by interval width
    (upper - lower) * mean_value

}
```

Let's define the function we want to integrate to be the Standard Normal curve:

![\frac{1}{\sqrt(2\pi)}e^{\frac{-x*x}{2}}](https://render.githubusercontent.com/render/math?math=%5Cfrac%7B1%7D%7B%5Csqrt(2%5Cpi)%7De%5E%7B%5Cfrac%7B-x*x%7D%7B2%7D%7D)

```
import scala.math._

// The function we want to integrate
def f(x: Double): Double = {
    math.exp(-x * x / 2) / math.sqrt(2 * Pi)
}
```

This is a probability distribution, symmetric about `0`, so it integrates to `1`.
Integrating this function between 0.0 and 5.0 should be approximately `0.5`.

We calculate the Monte Carlo integration estimate of the
area under this curve with 1000 Uniformly-distributed random variables.

```
val lb = 0.0
val ub = 5.0

val area = mc_integrate(fn=f, nsamp=1000, lower=lb, upper=ub)

println(area)
```
The output is
```
0.5298748535084071
```
which is not far off the true value of `0.5`.

Given the relative
simplicity of the algorithm, this is a decent result.
It's also good jumping off point for getting to grips
with more sophisiticated numerical integration methods.

The full code for this Scala project is [here](https://github.com/LeoKavanagh/mc-scala).