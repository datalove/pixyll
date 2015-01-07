---
layout:     post
title:      A touch of R
date:       2015-01-07 07:53:00
summary:    This is just a test to see how jekyll works with a feed specifically for R
categories: R
---

Just a test to see how well it deals with R and stuff.

So here's some R code.

{% highlight r %}
  x <- rnorm(100,100,10)
  y <- x + rnorm(100,5,1)
  mylm <- lm(x ~ y)
  plot(mylm)
{% endhighlight %}

And here's the output of some random R code at the console.

{% highlight rconsole %}
> python.call( "concat", a, b)
[1] 1 2 3 4 5 6 7 8
{% endhighlight %}
