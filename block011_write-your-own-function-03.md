---
title: "Write your own R functions, part 3"
output:
  html_document:
    toc: true
    toc_depth: 3
---



### Where were we? Where are we going?

In [part 2](block011_write-your-own-function-02.html) we generalized our first R function so it could take the difference between any two quantiles of a numeric vector. We also set default values for the underlying probabilities, so that, by default, we compute the max minus the min.

In this part, we tackle `NA`s, the special argument `...` and formal testing.

### Load the Gapminder data

As usual, load the Gapminder data.


```r
library(gapminder)
```

### Restore our max minus min function

Let's keep our previous function around as a baseline.


```r
qdiff3 <- function(x, probs = c(0, 1)) {
  stopifnot(is.numeric(x))
  the_quantiles <- quantile(x, probs)
  return(max(the_quantiles) - min(the_quantiles))
}
```

### Be proactive about `NA`s

I am being gentle by letting you practice with the Gapminder data. In real life, missing data will make your life a living hell. If you are lucky, it will be properly indicated by the special value `NA`, but don't hold your breath. Many built-in R functions have an `na.rm =` argument through which you can specify how you want to handle `NA`s. Typically the default value is `na.rm = FALSE` and typical default behavior is to either let `NA`s propagate or to raise an error. Let's see how `quantile()` handles `NA`s:


```r
z <- gapminder$lifeExp
z[3] <- NA
quantile(gapminder$lifeExp)
##      0%     25%     50%     75%    100% 
## 23.5990 48.1980 60.7125 70.8455 82.6030
quantile(z)
## Error in quantile.default(z): missing values and NaN's not allowed if 'na.rm' is FALSE
quantile(z, na.rm = TRUE)
##     0%    25%    50%    75%   100% 
## 23.599 48.228 60.765 70.846 82.603
```

So `quantile()` simply will not operate in the presence of `NA`s unless `na.rm = TRUE`. How shall we modify our function?

If we wanted to hardwire `na.rm = TRUE`, we could. Focus on our call to `quantile()` inside our function definition.


```r
qdiff4 <- function(x, probs = c(0, 1)) {
  stopifnot(is.numeric(x))
  the_quantiles <- quantile(x, probs, na.rm = TRUE)
  return(max(the_quantiles) - min(the_quantiles))
}
qdiff4(gapminder$lifeExp)
## [1] 59.004
qdiff4(z)
## [1] 59.004
```

This works but it is dangerous to invert the default behavior of a well-known built-in function and to provide the user with no way to override this.

We could add an `na.rm =` argument to our own function. We might even enforce our preferred default -- but at least we're giving the user a way to control the behavior around `NA`s.


```r
qdiff5 <- function(x, probs = c(0, 1), na.rm = TRUE) {
  stopifnot(is.numeric(x))
  the_quantiles <- quantile(x, probs, na.rm = na.rm)
  return(max(the_quantiles) - min(the_quantiles))
}
qdiff5(gapminder$lifeExp)
## [1] 59.004
qdiff5(z)
## [1] 59.004
qdiff5(z, na.rm = FALSE)
## Error in quantile.default(x, probs, na.rm = na.rm): missing values and NaN's not allowed if 'na.rm' is FALSE
```

### The useful but mysterious `...` argument

You probably could have lived a long and happy life without knowing there are at least 9 different algorithms for computing quantiles. [Go read about the `type` argument](http://www.rdocumentation.org/packages/stats/functions/quantile) of `quantile()`. TLDR: If a quantile is not unambiguously equal to an observed data point, you must somehow average two data points. You can weight this average different ways, depending on the rest of the data, and `type =` controls this.

Let's say we want to give the user of our function the ability to specify how the quantiles are computed, but we want to accomplish with as little fuss as possible. In fact, we don't even want to clutter our function's interface with this! This calls for the very special `...` argument. In English, this set of three dots is frequently called an "ellipsis".


```r
qdiff6 <- function(x, probs = c(0, 1), na.rm = TRUE, ...) {
  the_quantiles <- quantile(x = x, probs = probs, na.rm = na.rm, ...)
  return(max(the_quantiles) - min(the_quantiles))
}
```

The practical significance of the `type =` argument is virtually nonexistent, so we can't demo with the Gapminder data. Thanks to [\@wrathematics](https://twitter.com/wrathematics), here's a small example where we can (barely) detect a difference due to `type`.


```r
set.seed(1234)
z <- rnorm(10)
quantile(z, type = 1)
##         0%        25%        50%        75%       100% 
## -2.3456977 -0.8900378 -0.5644520  0.4291247  1.0844412
quantile(z, type = 4)
##        0%       25%       50%       75%      100% 
## -2.345698 -1.048552 -0.564452  0.353277  1.084441
all.equal(quantile(z, type = 1), quantile(z, type = 4))
## [1] "Mean relative difference: 0.1776594"
```

Now we can call our function, requesting that quantiles be computed in different ways.


```r
qdiff6(z, probs = c(0.25, 0.75), type = 1)
## [1] 1.319163
qdiff6(z, probs = c(0.25, 0.75), type = 4)
## [1] 1.401829
```

While the difference may be subtle, __it's there__. Marvel at the fact that we have passed `type = 1` through to `quantile()` *even though it was not a formal argument of our own function*.

The special argument `...` is very useful when you want the ability to pass arbitrary arguments down to another function, but without constantly expanding the formal arguments to your function. This leaves you with a less cluttered function definition and gives you future flexibility to specify these arguments only when you need to.

You will also encounter the `...` argument in many built-in functions -- read up [on `c()`](http://www.rdocumentation.org/packages/base/functions/c) or [`list()`](http://www.rdocumentation.org/packages/base/functions/list) -- and now you have a better sense of what it means. It is not a breezy "and so on and so forth."

There are also downsides to `...`, so use it with intention. In a package, you will have to work harder to create truly informative documentation for your user. Also, the quiet, absorbent properties of `...` mean it can sometimes silently swallow other named arguments, when the user has a typo in the name. Depending on whether or how this fails, it can be a little tricky to find out what went wrong.

### Use `testthat` for formal unit tests

Until now, we've relied on informal tests of our evolving function. If you are going to use a function alot, especially if it is part of a package, it is wise to use formal unit tests.

The [`testthat` package](https://github.com/hadley/testthat) provides excellent facilities for this, with a distinct emphasis on automated unit testing of entire packages. However, we can take it out for a test drive even with our one measly function.

We will construct a test with `test_that()` and, within it, we put one or more *expectations* that check actual against expected results. You simply harden your informal, interactive tests into formal unit tests. Here are some examples of tests and indicative expectations.


```r
library(testthat)

test_that('invalid args are detected', {
  expect_error(qdiff6("eggplants are purple"))
  expect_error(qdiff6(iris))
})

test_that('NA handling works', {
  expect_error(qdiff6(c(1:5, NA), na.rm = FALSE))
  expect_equal(qdiff6(c(1:5, NA)), 4)
})
```

No news is good news! Let's see what test failure would look like. Let's revert to a version of our function that does no `NA` handling, then test for proper `NA` handling. We can watch it fail.


```r
qdiff_no_NA <- function(x, probs = c(0, 1)) {
  the_quantiles <- quantile(x = x, probs = probs)
  return(max(the_quantiles) - min(the_quantiles))
}

test_that('NA handling works', {
  expect_that(qdiff_no_NA(c(1:5, NA)), equals(4))
})
## Error: Test failed: 'NA handling works'
## * missing values and NaN's not allowed if 'na.rm' is FALSE
## 1: expect_that(qdiff_no_NA(c(1:5, NA)), equals(4)) at <text>:7
## 2: condition(object)
## 3: expect_equal(x, expected, ..., expected.label = label)
## 4: quasi_label(enquo(object), label)
## 5: eval_bare(get_expr(quo), get_env(quo))
## 6: qdiff_no_NA(c(1:5, NA))
## 7: quantile(x = x, probs = probs) at <text>:2
## 8: quantile.default(x = x, probs = probs)
## 9: stop("missing values and NaN's not allowed if 'na.rm' is FALSE")
```

Similar to the advice to use assertions in data analytical scripts, I recommend you use unit tests to monitor the behavior of functions you (or others) will use often. If your tests cover the function's important behavior, then you can edit the internals freely. You'll rest easy in the knowledge that, if you broke anything important, the tests will fail and alert you to the problem. A function that is important enough for unit tests probably also belongs in a package, where there are obvious mechanisms for running the tests as part of overall package checks.

<!--

### other content

match.arg()

defaulting to NULL then checking is.null() and take it from there

-->

### Resources

Hadley Wickham's book [Advanced R](http://adv-r.had.co.nz)

  * Section on [function arguments](http://adv-r.had.co.nz/Functions.html#function-arguments)

Unit testing with `testthat`:

  * On [CRAN](https://cran.r-project.org/web/packages/testthat/index.html), development on [GitHub](https://github.com/hadley/testthat)

Hadley Wickham's [R packages](http://r-pkgs.had.co.nz) book

  * [Testing chapter](http://r-pkgs.had.co.nz/tests.html)
  
Article [testthat: Get Started with Testing](https://journal.r-project.org/archive/2011-1/RJournal_2011-1_Wickham.pdf) in The R Journal Vol. 3/1, June 2011. Maybe this is completely superceded by the newer chapter above? Be aware that parts could be out of date, but I recall it was a helpful read.
