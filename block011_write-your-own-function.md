# Write your own R functions



### What and why?

My goal here is to reveal the __process__ many long-time useRs employ for writing functions. I also want to illustrate why we work the way we do. Merely looking at the finished product, e.g. source code for R packages, can be extremely deceiving. Reality is generally much uglier ... but more interesting!

Why are we covering this now, in the midst of our data aggregation work? Powerful machines like `dplyr`, `plyr`, and even the built-in `apply` family of functions are ready and waiting to apply your purpose-built functions to various bits of your data. If you can express your analytical wishes in a function, these tools will make you extremely powerful with data.

### Load the Gapminder data

As usual, load the Gapminer excerpt.


```r
gDat <- read.delim("gapminderDataFiveYear.txt")
str(gDat)
## 'data.frame':	1704 obs. of  6 variables:
##  $ country  : Factor w/ 142 levels "Afghanistan",..: 1 1 1 1 1 1 1 1 1 1 ...
##  $ year     : int  1952 1957 1962 1967 1972 1977 1982 1987 1992 1997 ...
##  $ pop      : num  8425333 9240934 10267083 11537966 13079460 ...
##  $ continent: Factor w/ 5 levels "Africa","Americas",..: 3 3 3 3 3 3 3 3 3 3 ...
##  $ lifeExp  : num  28.8 30.3 32 34 36.1 ...
##  $ gdpPercap: num  779 821 853 836 740 ...
## or do this if the file isn't lying around already
## gd_url <- "http://tiny.cc/gapminder"
## gDat <- read.delim(gd_url)
```

### Task we aim to function-ize

Say you've got a numeric vector. Compute the difference between its min and max. `lifeExp` or `pop` or `gdpPercap` are great examples of a typical input. You can imagine wanting to get this statistic after we slice up the Gapminder data by year, country, continent, or combinations thereof.

### Get something that works

First, develop some working code for interactive use, using a representative input. I'm going to use Gapminder's life expectancy variable.

R functions that will be useful: `min()`, `max()`, `range()`. __Read their documentation.__


```r
## get to know the functions mentioned above
min(gDat$lifeExp)
## [1] 23.599
max(gDat$lifeExp)
## [1] 82.603
range(gDat$lifeExp)
## [1] 23.599 82.603

## some natural solutions
max(gDat$lifeExp) - min(gDat$lifeExp)
## [1] 59.004
with(gDat, max(lifeExp) - min(lifeExp))
## [1] 59.004
range(gDat$lifeExp)[2] - range(gDat$lifeExp)[1]
## [1] 59.004
with(gDat, range(lifeExp)[2] - range(lifeExp)[1])
## [1] 59.004
diff(range(gDat$lifeExp))
## [1] 59.004
```

Internalize this "answer" because our informal testing relies on you noticing departures from this.

#### Skateboard >> perfectly formed rear-view mirror

This image [widely attributed to the Spotify development team](http://blog.fastmonkeys.com/?utm_content=bufferc2d6e&utm_medium=social&utm_source=twitter.com&utm_campaign=buffer) conveys an important point.

![alt text](img/spotify-howtobuildmvp.gif)

Build that skateboard before you build the car or some fancy car part. A limited-but-functioning thing is very useful. It also keeps the spirits high.

This is related to the valuable [Telescope Rule](http://c2.com/cgi/wiki?TelescopeRule):

> It is faster to make a four-inch mirror then a six-inch mirror than to make a six-inch mirror.

### Turn the working interactive code into a function and re-test

Add NO new functionality! Just write your very first R function.


```r
max_minus_min <- function(x) max(x) - min(x)
max_minus_min(gDat$lifeExp)
## [1] 59.004
```

Check that you're getting the same answer as you did with your interactive code. Test it eyeball-o-metrically at this point.

### Test your function

#### Test on new inputs

Pick some new articial inputs where you know (at least approximately) what your function should return.


```r
max_minus_min(1:10)
## [1] 9
max_minus_min(runif(1000))
## [1] 0.9947266
```

I know that 10 minus 1 is 9. I know that random uniform [0, 1] variates will be between 0 and 1. Therefore max - min should be less than 1. If I take LOTS of them, max - min should be pretty close to 1.

It is intentional that I tested on integer input as well as floating point. Likewise, I like to use valid-but-random data for this sort of check.

#### Test on real data but *different* real data

Back to the real world now. Two other quantitative variables are lying around: `gdpPercap` and `pop`. Let's have a go.


```r
max_minus_min(gDat$gdpPercap)
## [1] 113282
max_minus_min(gDat$pop)
## [1] 1318623085
```

Either check these results "by hand" or apply the "does that even make sense?" test.

#### Test on weird stuff

Now we are going to try to break our function. Don't get truly diabolical (yet). Just make the kind of mistakes you can imagine yourself making at 2am when you rediscover this useful function you wrote 3 years ago.


```r
max_minus_min(gDat) ## hey sometimes things "just work" on data.frames!
## Error: only defined on a data frame with all numeric variables
max_minus_min(gDat$country) ## factors are kind of like integer vectors, no?
## Error: max not meaningful for factors
max_minus_min("eggplants are purple") ## i have no excuse for this one
## Error: non-numeric argument to binary operator
```

How happy are you with those error messages? You must imagine that some entire __script__ has failed because the user was hoping to just source it without re-reading it. If a colleague or future you encountered that, how hard is it to pinpoint the usage problem? Will you just give up and say "this script/function doesn't work"?

#### I will scare you now

Here are some great examples STAT545 students devised during class where the function __should break but it does not.__


```r
max_minus_min(gDat[c('lifeExp', 'gdpPercap', 'pop')])
## [1] 1318683072
max_minus_min(c(TRUE, TRUE, FALSE, TRUE, TRUE))
## [1] 1
```

In both cases, R's eagerness to make sense of our requests is unfortunately successful. In the first case, a data.frame containing just the quantitative variables is eventually coerced into numeric vector. We can compute max minus min, even though it makes absolutely no sense at all. In the second case, a logical vector is converted to zeroes and ones, which might merit an error or at least a warning.

### Check the validity of arguments

#### stopifnot

For functions that will be used again -- which is not all of them! -- it is good to check the validity of arguments.

`stopifnot()` is the entry level solution. I use it here to make sure the input `x` is a numeric vector.


```r
mmm <- function(x) {
  stopifnot(is.numeric(x))
  max(x) - min(x)
  }
mmm(gDat)
## Error: is.numeric(x) is not TRUE
mmm(gDat$country)
## Error: is.numeric(x) is not TRUE
mmm("eggplants are purple")
## Error: is.numeric(x) is not TRUE
mmm(gDat[c('lifeExp', 'gdpPercap', 'pop')])
## Error: is.numeric(x) is not TRUE
mmm(c(TRUE, TRUE, FALSE, TRUE, TRUE))
## Error: is.numeric(x) is not TRUE
```

And we see that it catches all of the mistakes we're trying to guard against (so far).

#### if then stop

`stopifnot()` doesn't provide a very good error message. The next approach is very widely used. Put your validity check inside an `if()` statement and call `stop()` yourself, with a custom error message, in the body.


```r
mmm2 <- function(x) {
  if(!is.numeric(x)) {
    stop('I am so sorry, but this function only works for numeric input!')
  }
  max(x) - min(x)
  }
mmm2(gDat)
## Error: I am so sorry, but this function only works for numeric input!
```

In addition to offering an apology, note the error raised also contains helpful info on *which* function threw the error. Nice touch.

*Note: the above is true when run interactively but not true in the rendered document. I am getting to the bottom of that.*

### Packages for formal checks at run time

The [`assertthat` package](https://github.com/hadley/assertthat) "provides a drop in replacement for `stopifnot()`." That is quite literally true. The function `mmm3` differs from `mmm2` only in the replacement of `stopifnot()` by `assert_that()`.


```r
## install if you do not already have!
## install.packages(assertthat)
library(assertthat)
mmm3 <- function(x) {
  assert_that(is.numeric(x))
  max(x) - min(x)
  }
mmm3(gDat)
## Error: x is not a numeric or integer vector
```

The [`ensurer` package](https://github.com/smbache/ensurer) is another, newer package with some similar goals, so you may want to check that out as well.



#### Sidebar: other uses for `assertthat` or `ensurer`

Another good use of these packages is to leave checks behind in data analytical scripts . Consider our repetitive use of Gapminder. Every time we load this data, we inspect it, e.g., with `str()`. Informally, we're checking that is still has 1704 rows. But we could, and probably should, formalize that with a call like `assert_that(nrow(gDat) == 1704)`. This would tell us if the data suddenly changed, alerting us to a problem with the data file or the import.

### Generalize our function to other quantiles

The max and the min are special cases of a __quantile__. Here are other special cases you may have heard of:

  * median = 0.5 quantile
  * 1st quartile = 0.25 quantile
  * 3rd quartile = 0.75 quantile
  
If you're familiar with [box plots](http://en.wikipedia.org/wiki/Box_plot), the rectangle typically runs from the 1st quartile to the 3rd quartile, with a line at the median.

If $q$ is the $p$-th quantile of a set of $n$ observations, what does that mean? Approximately $pn$ of the observations are less than $q$ and $(1 - p)n$ are greater than $q$. Yeah, you need to worry about rounding to an integer and less/greater than or equal to, but these details aren't critical here.

Let's generalize our function to take the difference between any two quantiles. We can still consider the max and min, if we like, but we're not limited to that.

#### Get something that works, again

The eventual inputs to our new function will be the data `x` and two probabilities.

First, play around with the `quantile()` function. Convince yourself you know how to use it.


```r
quantile(gDat$lifeExp)
##      0%     25%     50%     75%    100% 
## 23.5990 48.1980 60.7125 70.8455 82.6030
quantile(gDat$lifeExp, probs = 0.5)
##     50% 
## 60.7125
median(gDat$lifeExp)
## [1] 60.7125
quantile(gDat$lifeExp, probs = c(0.25, 0.75))
##     25%     75% 
## 48.1980 70.8455
boxplot(gDat$lifeExp, plot = FALSE)$stats
##         [,1]
## [1,] 23.5990
## [2,] 48.1850
## [3,] 60.7125
## [4,] 70.8460
## [5,] 82.6030
```

Now write a code snippet that takes the difference between two quantiles.


```r
the_probs <- c(0.25, 0.75)
the_quantiles <- quantile(gDat$lifeExp, probs = the_probs)
max(the_quantiles) - min(the_quantiles)
## [1] 22.6475
IQR(gDat$lifeExp) # hey, we've reinvented IQR
## [1] 22.6475
```

Since I knew the difference between the 1st and 3rd quartiles is the interquartile range, I used this opportunity to informally check myself against a built-in function, `IQR()`.

#### Turn the working interactive code into a function and re-test

I'll use `qdiff` as the base of our function's name. I copy the overall structure from our previous `mmm` attempts but replace the "guts" of it with the more general code we just developed.


```r
qdiff1 <- function(x, probs) {
  assert_that(is.numeric(x))
  the_quantiles <- quantile(x = x, probs = probs)
  max(the_quantiles) - min(the_quantiles)
}
qdiff1(gDat$lifeExp, probs = c(0.25, 0.75))
## [1] 22.6475
qdiff1(gDat$lifeExp, probs = c(0, 1))
## [1] 59.004
max_minus_min(gDat$lifeExp)
## [1] 59.004
```

Again we do some informal tests against familiar results.

### Argument names: freedom and conventions

I can name my arguments anything I like. Proof:


```r
qdiff2 <- function(zeus, hera) {
  assert_that(is.numeric(zeus))
  the_quantiles <- quantile(x = zeus, probs = hera)
  return(max(the_quantiles) - min(the_quantiles))
}
qdiff2(zeus = gDat$lifeExp, hera = 0:1)
## [1] 59.004
```

While I can name my arguments after Greek gods, it's usually a bad idea. Take all opportunities to make things more self-explanatory via meaningful names.

This is better:


```r
qdiff3 <- function(my_x, my_probs) {
  assert_that(is.numeric(my_x))
  the_quantiles <- quantile(x = my_x, probs = my_probs)
  return(max(the_quantiles) - min(the_quantiles))
}
qdiff3(my_x = gDat$lifeExp, my_probs = 0:1)
## [1] 59.004
```

If you are going to pass the arguments of your function as arguments of a built-in function, consider copying the argument names. Again, the reason is to reduce your cognitive load. This is what I did in our first iteration of `qdiff`:


```r
qdiff1
## function(x, probs) {
##   assert_that(is.numeric(x))
##   the_quantiles <- quantile(x = x, probs = probs)
##   max(the_quantiles) - min(the_quantiles)
## }
```

We took this detour so you could see there is no *structural* relationship between my arguments (`x` and `probs`) and those of `quantile()` (also `x` and `probs`). The similarity or equivalence of the names __accomplishes nothing__ as far as R is concerned; it is solely for the benefit of humans reading, writing, and using the code. Which actually matters.

### Default values: freedom to NOT specify the arguments

What happens if we call our function but neglect to specify the probabilities?


```r
qdiff1(gDat$lifeExp) ## oops! must specify probs argument
## Error: argument "probs" is missing, with no default
```

It is often nice to provide some reasonable default values, so users don't have to specify everything, all the time.

We started by focusing on the max and the min, so I think those make reasonable defaults. Here's how to specify that in a function definition.


```r
qdiff4 <- function(x, probs = c(0, 1)) {
  assert_that(is.numeric(x))
  the_quantiles <- quantile(x, probs)
  return(max(the_quantiles) - min(the_quantiles))
}
```

Again we check how the function works, in old examples and new.


```r
qdiff4(gDat$lifeExp)
## [1] 59.004
max_minus_min(gDat$lifeExp)
## [1] 59.004
qdiff4(gDat$lifeExp, c(0.1, 0.9))
## [1] 33.5862
```

### Check the validity of arguments, again

EXERCISE: upgrade our arguement validity checks in light of the new argument `probs`


```r
## we're not checking that probs is numeric
## we're not checking that probs is length 2
## we're not checking that probs are in [0,1]
```
