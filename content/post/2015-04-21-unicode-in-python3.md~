---
layout: post
title: Unicode in python3
category: [tech]
tags: [python]
---

Long time since I touched python last time. Today I tried to write a line of code like this:
{% highlight python %}
message = "光臨"
print(message)
{% endhighlight %}

And I run this program with python3, and get something like this: ```UnicodeEncodeError: 'ascii' codec can't encode characters```.

It's weird because in python3 every str is default to unicode.

After some google, I find that ```print(sys.stdout.encoding)``` gives me ```US-ASCII```, which seems to explain why ascii appears in the error message.
So I set $LANG with ```export LANG=en_US.UTF-8```, then ```sys.stdout.encoding``` becomes ```UTF-8```, and ```光臨``` get output. Here, the encoding of my
terminal is also utf-8.

It seems that, when doing output to stdio, python will decode the string with sys.stdout.encoding which seems determined by ```$LANG``` or ```$LC_XXX```. When $LANG is not set,
it will be ASCII, of couse this will result in decode error.

Also note that, terminal encoding another different thing. It determines how bytes are rendered in terminal. If utf-8 is set, then terminal will assume the byte codes are
utf-8 streams, and will encode them acording to utf-8. If byte codes is actully other codings, then funny characters will appear on terminal.
