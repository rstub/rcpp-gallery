---
title: 'RcppNumerical&#58; Numerical integration and optimization with Rcpp'
author: "Yixuan Qiu and Ralf Stubner"
license: GPL (>= 2)
mathjax: true
tags: basics 
summary: "Introduces RcppNumerical, a library of numerical integration and optimization methods."
layout: post
src: 2019-08-07-rcppnumerical-intro.Rmd
---



## Introduction

[Rcpp](https://CRAN.R-project.org/package=Rcpp) is a
powerful tool to write fast C++ code to speed up R programs. However,
it is not easy, or at least not straightforward, to compute numerical
integration or do optimization using pure C++ code inside Rcpp, even though
R has the functions `Rdqags`, `Rdqagi`, `nmmin`, `vmmin` etc. in its API
to accomplish such tasks.

**RcppNumerical** integrates a number of open source numerical computing
libraries into Rcpp, so that users can call convenient functions to
accomplish such tasks.


- To use `RcppNumerical` with `Rcpp::sourceCpp()`, add the following two lines
to the C++ source file:

    ```cpp
    // [[Rcpp::depends(RcppEigen)]]
    // [[Rcpp::depends(RcppNumerical)]]
    ```

- To use `RcppNumerical` in your package, add the corresponding fields to the
`DESCRIPTION` file:

    ```
    Imports: RcppNumerical
    LinkingTo: Rcpp, RcppEigen, RcppNumerical
    ```

    Also in the `NAMESPACE` file, add:

    ```
    import(RcppNumerical)
    ```

## Numerical Integration

### One-dimensional

The one-dimensional numerical integration code contained in **RcppNumerical**
is based on the [NumericalIntegration](https://github.com/tbs1980/NumericalIntegration)
library developed by [Sreekumar Thaithara Balan](https://github.com/tbs1980),
[Mark Sauder](https://github.com/mcsauder), and Matt Beall.

To compute integration of a function, first define a functor derived from
the `Func` class (under the namespace `Numer`):

```cpp
class Func
{
public:
    virtual double operator()(const double& x) const = 0;
    virtual void eval(double* x, const int n) const
    {
        for(int i = 0; i < n; i++)
            x[i] = this->operator()(x[i]);
    }
    
    virtual ~Func() {}
};
```

The first function evaluates one point at a time, and the second version
overwrites each point in the array by the corresponding function values.
Only the second function will be used by the integration code, but usually it
is easier to implement the first one.

**RcppNumerical** provides a wrapper function for the **NumericalIntegration**
library with the following interface:

```cpp
inline double integrate(
    const Func& f, double& lower, double& upper,
    double& err_est, int& err_code,
    const int subdiv = 100, const double& eps_abs = 1e-8, const double& eps_rel = 1e-6,
    const Integrator<double>::QuadratureRule rule = Integrator<double>::GaussKronrod41
)
```


See the [README](https://github.com/yixuan/RcppNumerical) page for the
explanation of each argument. Below shows an example that calculates the 
integral of 

$$f(x) = \frac{1}{(x+1) \sqrt{x}}$$

This function is mentioned on `?integrate` as an example that converges very slowly when one tries to calculate

$$ \int_0^\infty \mathrm{d}x \frac{1}{(x+1) \sqrt{x}} = \pi$$
by using successively larger but finite upper bounds as oppsed to using `Inf` as upper bound directly:


{% highlight r %}
integrand <- function(x) {1/((x+1)*sqrt(x))}
upper <- c(10^(1:6), Inf)
values <- t(sapply(upper, integrate, f = integrand, lower = 0, stop.on.error = FALSE))
values <- cbind(upper, values)
knitr::kable(values[, c("upper", "value", "abs.error", "message")])
{% endhighlight %}



|upper |value                |abs.error            |message                            |
|:-----|:--------------------|:--------------------|:----------------------------------|
|10    |2.52903787970396     |0.000300103104891036 |OK                                 |
|100   |2.94225520951919     |5.13700863358224e-06 |OK                                 |
|1000  |3.07836803018286     |0.00026653890881434  |OK                                 |
|10000 |3.12159313933017     |0.000184650531104946 |OK                                 |
|1e+05 |3.13526812233686     |4.19228987968978e-07 |OK                                 |
|1e+06 |-0.00200117491927485 |2.41562914350211e-05 |the integral is probably divergent |
|Inf   |3.14159265302791     |2.65149992615399e-05 |OK                                 |

In order to calculate this integral with **RcppNumerical** we need a functor deriving from `Numer::Func` for the mathematical function. For convenience we also use a wrapper function to expose the functionality to R:


{% highlight cpp %}
// [[Rcpp::depends(RcppEigen)]]
// [[Rcpp::depends(RcppNumerical)]]
#include <RcppNumerical.h>

class integrand : public Numer::Func
{
public:
    integrand() {}
    
    double operator()(const double& x) const
    {
        return 1.0/((x + 1.0) * std::sqrt(x));
    }
};

// [[Rcpp::export]]
double integral(double a, double b)
{
    integrand f;
    double err_est;
    int err_code;
    double value = Numer::integrate(f, a, b, err_est, err_code);
    if (err_code != 0)
        Rcpp::stop("Numerical integration failed!");
    return value;
}
{% endhighlight %}

Calculating the integral for fixed $$a=0$$ and increasing $$b$$ we get:


{% highlight r %}
b <- 10^(1:10)
Fab <- sapply(b, integral, a = 0)
ggplot2::qplot(b, Fab, geom = "line", log = "x", xlab = "b", ylab = "F(0,b)")
{% endhighlight %}

![plot of chunk unnamed-chunk-3](../figure/2019-08-07-rcppnumerical-intro-unnamed-chunk-3-1.png)

So it seems to be converging nicely, but eventually breaks down here as well:


{% highlight r %}
tryCatch(integral(0, 1e19), error=print)
{% endhighlight %}



<pre class="output">
&lt;Rcpp::exception in integral(0, 1e+19): Numerical integration failed!&gt;
</pre>

Until recently it was not possible to explicitly use `Inf` as upper bound with **RcppNumerical**. A recent pull request changed that for the development version, though. The basic idea for handling integration with infinite bounds is to use varaible substitution to map the integral to a finite range. In the case where both limits are infinite, we use $$x = (1-t)/t$$ resulting in

$$
\int_{-\infty}^{\infty} \mathrm{d}x f(x) = \int_0^1 \mathrm{d}t \frac{f((1-t)/t) + f(-(1-t)/t)}{t^2}
$$

If only one of the limits is infinite, we use the substitutions $$x = a + (1-t)/t$$ and $$x = b - (1-t)/t$$ resulting in

$$
\int_{a}^{\infty} \mathrm{d}x f(x) = \int_0^1 \mathrm{d}t \frac{f(a+(1-t)/t)}{t^2}
$$

and 

$$
\int_{-\infty}^{b} \mathrm{d}x f(x) = \int_0^1 \mathrm{d}t \frac{f(b-(1-t)/t)}{t^2}
$$

These variable substitutions are implemented in a templated functor that uses the original functor as template argument:

```cpp
template<class T>
class transform_infinite: public Func
{
private:
    T func;
    double lower;
    double upper;
public:
    transform_infinite(T _func, double _lower, double _upper) :
    func(_func), lower(_lower), upper(_upper) {}

    double operator() (const double& t) const
    {
        double x = (1 - t) / t;
        bool upper_finite = (upper <  std::numeric_limits<double>::infinity());
        bool lower_finite = (lower > -std::numeric_limits<double>::infinity());
        if (upper_finite && lower_finite)
            Rcpp::stop("At least on limit must be infinite.");
        else if (lower_finite)
            return func(lower + x) / pow(t, 2);
        else if (upper_finite)
            return func(upper - x) / pow(t, 2);
        else
            return (func(x) + func(-x)) / pow(t, 2);
    }
};

```

This templated functor can than be used to convert the original functor, e.g.:

```cpp
integrand f;
transform_infinite<integrand> g(f, upper, lower);
double err_est;
int err_code;
double value = integrate(g, 0.0, 1.0, err_est, err_code);
```

However, one does not have to deal with this directly, since `Numer::integrate` handles this already:


{% highlight r %}
integral(0, Inf)
{% endhighlight %}



<pre class="output">
[1] 3.141593
</pre>

### Multi-dimensional

Multi-dimensional integration in **RcppNumerical** is done by the
[Cuba](http://www.feynarts.de/cuba/) library developed by
[Thomas Hahn](http://wwwth.mpp.mpg.de/members/hahn/).

To calculate the integration of a multivariate function, one needs to define
a functor that inherits from the `MFunc` class:

```cpp
class MFunc
{
public:
    virtual double operator()(Constvec& x) = 0;
    
    virtual ~MFunc() {}
};
```

Here `Constvec` represents a read-only vector with the definition

```cpp
// Constant reference to a vector
typedef const Eigen::Ref<const Eigen::VectorXd> Constvec;
```

(Basically you can treat `Constvec` as a `const Eigen::VectorXd`. Using
`Eigen::Ref` is mainly to avoid memory copy. See the explanation
[here](http://eigen.tuxfamily.org/dox/classEigen_1_1Ref.html).)

The function provided by **RcppNumerical** for multi-dimensional
integration is

```cpp
inline double integrate(
    MFunc& f, Constvec& lower, Constvec& upper,
    double& err_est, int& err_code,
    const int maxeval = 1000,
    const double& eps_abs = 1e-6, const double& eps_rel = 1e-6
)
```

See the [README](https://github.com/yixuan/RcppNumerical) page for the
explanation of each argument and a detailed example.


## Numerical Optimization

Currently **RcppNumerical** contains the L-BFGS algorithm for unconstrained
minimization problems based on the
[LBFGS++](https://github.com/yixuan/LBFGSpp) library.

Again, one needs to first define a functor to represent the multivariate
function to be minimized.

```cpp
class MFuncGrad
{
public:
    virtual double f_grad(Constvec& x, Refvec grad) = 0;
    
    virtual ~MFuncGrad() {}
};
```

Same as the case in multi-dimensional integration, `Constvec` represents a
read-only vector and `Refvec` a writable vector. Their definitions are

```cpp
// Reference to a vector
typedef Eigen::Ref<Eigen::VectorXd>             Refvec;
typedef const Eigen::Ref<const Eigen::VectorXd> Constvec;
```

The `f_grad()` member function returns the function value on vector `x`,
and overwrites `grad` by the gradient.

The wrapper function for **LBFGS++** is

```cpp
inline int optim_lbfgs(
    MFuncGrad& f, Refvec x, double& fx_opt,
    const int maxit = 300, const double& eps_f = 1e-6, const double& eps_g = 1e-5
)
```

See the [README](https://github.com/yixuan/RcppNumerical) page for the
explanation of each argument and a detailed example.