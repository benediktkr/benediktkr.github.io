---
layout: post
title:  "Adding a date filter to bottle.py"
date:   2016-05-07 14:36:17 +0200
categories: dev
---

[Bottle.py](http://bottlepy.org) is a simple micro web-framework for
Python. Out of the box, it supports input validation by "wildcard
filters" for integers, floating point, paths and custom regular
expressions.

# Validating ISO formatted dates

Dates are a common input parameter that can be tricky to
validate. Reading the
[Bottle documentation](http://bottlepy.org/docs/dev/routing.html#wildcard-filters)
we learn that you can define your own filters for input
validation. Great, let's do that.

The problem here is that the filter is supposed to
[use regular expressions](https://xkcd.com/208/). That is
[just as horrible](http://stackoverflow.com/a/15504877/386827) as it
sounds. So let's not do that.

With Python, you are encouraged to be a little rude to your interpreter and follow the ["it's easier to ask for forgivess, than to ask for permission"](https://docs.python.org/2/glossary.html#term-eafp)-approach.

In other words, you shouldn't *check* if the input is valid and then
handle it, you should assume it is and then handle the exception if
it's not.

Unfortunately, the filters in Bottle.py assume the opposite. Luckily,
when the filter fails, Bottle will automatically return a `HTTP 400`
error to the client. This is correct behaviour, since `400 Bad
Request`
[means that the server won't handle the request because the client made an erroneous request](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes#4xx_Client_Error).

With that in mind, let's construct a filter with a regular expression
that matches anything and then attempts to read a date and failing if
it isn't. It's a little bit hacky, but whatever --- you're a
programmer (right?).

{% highlight python %}
from datetime import datetime

def date_filter(frmt):
    regex = r".*"

    def to_python(match):
        return datetime.strptime(match, frmt or "%Y-%m-%d").date()

    def to_url(date):
        return date.isoformat()

    return regex, to_python, to_url
{% endhighlight %}

If the input is not a date, then `datetime.strptime` will raise a `ValueError` and Bottle will happily give the client a `400` error mesage. Great!
