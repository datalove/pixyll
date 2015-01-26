---
layout:     post
title:      dplyr in <100 lines (filter)
date:       2015-01-25
summary:    After I picked up a copy of Hadley Wickham's amazing Advanced R book, I wanted a pet project to apply some of what I learned.
categories: r dplyr
---

If you've been around R for a little while, you've most likely heard
of Hadley Wickham. His amazing packages (including **ggplot2**, **reshape2**,
**plyr** and many more) have shaped the R ecosystem into what it is today.
Without them, it's unlikely I would have continued learning R.

One of Hadley's latest packages is called **dplyr**, and it's an absolute game-changer.
I was giddy with excitement the first time I saw a demonstration. It
lets you perform almost any data manipulation task as a series of
five simple verbs; {% ihighlight r%}mutate{% endihighlight %}, `filter`, `arrange`, `group_by` and `summarise`.

A lot of effort was put into making it as fast as possible, while
maintaining an incredibly simple syntax. In most cases, it approaches
the speed of the amazing data.table package, but with a much more
approachable syntax. And to boot, it lets us transparently use
out-of-memory (i.e. in-database) data as if it were a local dataframe.

Fast forward a few months and Hadley's new R book, Advanced R, was
being updated on his website as he wrote it. It wasn't until a print
copy of his book was available that I put serious effort into going
through the content. And I'm glad to say it's as thorough and well
written as we've come to expect.

I've wanted to peer under the hood of dplyr for a while to figure
out how it works, but to a intermediate R programmer like myself,
the codebase was a bit too complex.

Instead of using dplyr's codebase to figure out how dplyr works,
I decided to use what I was learning in Advanced R to write my own
version, using only base R function.

Turns out all that's needed is about 70 lines of code.

It won't be nearly as fast as dplyr, or nearly as robust as dplyr,
but it could look and smell like dplyr.

### Replicating `filter`

The most common way to subset a data frame in R is to use the `[]` function.

If you wanted to see only cars form the **mtcars** data set, you do something like
this.  

{% highlight rconsole %}
> mtcars[mpg > 30,]
Error in `[.data.frame`(mtcars, mpg > 30, ) : object 'mpg' not found
{% endhighlight %}

Oops! We forgot to tell R that `mpg` is a column in `mtcars`, and not a variable
in our environment.

{% highlight rconsole %}
> mtcars[mtcars$mpg > 30,]
mpg cyl disp  hp drat    wt  qsec vs am gear carb
Fiat 128       32.4   4 78.7  66 4.08 2.200 19.47  1  1    4    1
Honda Civic    30.4   4 75.7  52 4.93 1.615 18.52  1  1    4    2
Toyota Corolla 33.9   4 71.1  65 4.22 1.835 19.90  1  1    4    1
Lotus Europa   30.4   4 95.1 113 3.77 1.513 16.90  1  1    5    2
{% endhighlight %}

Now we just need to wrap that into a function.

{% highlight rconsole %}
> filter2 <- function(data,expr) {data[expr,]}
> filter2(mtcars, mpg > 30)
Error in `[.data.frame`(data, expr, ) : object 'mpg' not found
{% endhighlight %}

We get the error we had previously when trying to filter from the console. Let's back it up a bit, and redefine our function to just print our filter expression.

{% highlight rconsole %}
> filter2 <- function(data,expr) {print(expr)}
> filter2(mtcars, mpg > 30)
Error in print(expr) : object 'mpg' not found
In addition: Warning message:
In print(expr) : restarting interrupted promise evaluation
{% endhighlight %}

No love there. What's going on?

 It turns out that every time we try to do anything with our `expr` variable, R
evaluates it on the spot. We need to do something to keep to stop it from doing
that, and the answer is the `substitute()` function.

`substitute()` is a special primitive function that does not evaluate its
arguments, and returns what's known as a `call` object.

We still want our filter expression evaluated though, so we'll use the `eval()`
function to evaluate our expression turned `call`. But don't forget that we'll
need to evaluate it the data we've passed into the function.

{% highlight rconsole %}
> filter2 <- function(data,expr) { data[eval(substitute(expr),data), ]}
> filter2(mtcars, mpg > 30)
mpg cyl disp  hp drat    wt  qsec vs am gear carb
Fiat 128       32.4   4 78.7  66 4.08 2.200 19.47  1  1    4    1
Honda Civic    30.4   4 75.7  52 4.93 1.615 18.52  1  1    4    2
Toyota Corolla 33.9   4 71.1  65 4.22 1.835 19.90  1  1    4    1
Lotus Europa   30.4   4 95.1 113 3.77 1.513 16.90  1  1    5    2
{% endhighlight %}

Success! Our `filter2()` works as expected. The only problem is that it only
works for a single filter expression, unlike `dplyr::filter`, which allows us to
the following.
{% highlight rconsole %}
> dplyr::filter(mtcars, mpg > 30, carb == 2)
mpg cyl disp  hp drat    wt  qsec vs am gear carb
1 30.4   4 75.7  52 4.93 1.615 18.52  1  1    4    2
2 30.4   4 95.1 113 3.77 1.513 16.90  1  1    5    2
{% endhighlight %}

What changes do we need to make to `filter2()`?

Adding a `...` argument to our function will allow us to specify an arbitrary
number of filtering expressions. A useful function to use with `...` is
`alist()`, which simply puts all of its arguments into a list without evaluating
them. As before, we'll stil want to `eval` and `substitute` as before. We can
create a list of filter expressions to evaluate like so `arg <-
eval(substitute(alist(...)))`.

Then we can access each filter expression supplied by the user by simply
subsetting arg like `arg[[1]]` or `arg[[2]]`. What we now want to do is
repeatedly filter our data for each item in the `arg` list.

In most situations in R, we want to use `lapply` to do something to each element
in a list. But in this case, we can't use it since we want pass the filtered
results from each expression to the next. Here, a `for` loop is most
appropriate.
That leads us to our final function.

{% highlight r %}
filter2 <- function(.data, ...) {

  expr <- eval(substitute(alist(...)))

  for(i in seq_along(expr)) {
    r <- expr[[i]]  
    .data <- .data[eval(r,.data), ]
  }  
  .data
}
{% endhighlight %}

{% highlight rconsole %}
> filter2(mtcars, mpg > 30, carb == 2)
mpg cyl disp  hp drat    wt  qsec vs am gear carb
Honda Civic  30.4   4 75.7  52 4.93 1.615 18.52  1  1    4    2
Lotus Europa 30.4   4 95.1 113 3.77 1.513 16.90  1  1    5    2
{% endhighlight %}

And pleasingly, we don't need anything extra to cater for more complex
filter expressions like the ones below, which filters out the bottom 80% least
fuel efficient cars, and keeps only those whose displacement per cyclinder
exceeds 30.

{% highlight rconsole %}
> filter2(mtcars, mpg > quantile(mpg, 0.80), disp/cyl > 30)
mpg           cyl  disp hp drat   wt qsec vs am gear carb
Merc 240D     24.4   4 146.7 62 3.69 3.19 20.0  1  0    4    2
Porsche 914-2 26.0   4 120.3 91 4.43 2.14 16.7  0  1    5    2
{% endhighlight %}

In the next section, we'll build `mutate2()`.
