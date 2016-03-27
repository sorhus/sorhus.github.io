---
layout: post
title:  "bash-reduce part 1"
---
Almost 2 years ago now, I wrote a toy project: a map-reduce framework in bash and
awk. I was working a lot with [Hadoop](http://hadoop.apache.org/) map-reduce and
[Scalding](https://github.com/twitter/scalding) at the time, and a colleague
had reimplemented a few common command line tools in Scalding. That enabled him
to type something like this if he wanted to look for the word `HELP` in `/my/file`

{% highlight bash %}
hadoop jar my.jar com.twitter.scalding.Tool CMDTools --hdfs --input /my/file --grep HELP
{% endhighlight %}
<br>

The command would spawn a map-reduce job on our cluster and give him the results.
I thought it was a fun idea, and it inspired to me do something similar.

What I did was to implement a map-reduce framework in bash. To call it a framework
seems a bit over the top, as it's really just a few short scripts. But it does
enable me to write a map function and a reduce function in pure awk, and I can
distribute the workload over multiple cpus on my laptop, or even over many
computers. It doesn't really have a practical use case, but it's fun!

# Compulsory word count example
As anyone that ever looked at a map-reduce tutorial will know, word count is
the canonical first problem to solve when trying a new map-reduce framework. Word
count is the problem of finding the number of times each word occurs in a body
of text. Typically the body of text is large, which is why we would need a
distributed framework to compute it, rather than just asking our text editor for it.

### map-reduce
As a brief background, what map-reduce does is to split up the task in many small
parts. Each part can then be executed by a separate process, preferably in parallel.
The way it splits the task is typically by reading the input one line at a time,
and outputting a sequence of key-value pairs. This is called the map phase
(I guess it's actually a flatmap!). The framework will then take these key-value
pairs, sort them by the key, and apply a reduce function to all the values of the
same key. To summarize, we can describe the functions in a scala like syntax as

{% highlight scala %}
def map(input: I): List[(key: K, value: V)]

def reduce(key: K, values: List[V]): R
{% endhighlight %}
<br>

The main point is that to the author of these functions, the only concern is how
to process one unit of input in the mapper and how to process all the values for
a given key in the reducer. It hides so much complexity and I guess that's the
main reason why Hadoop has had such a massive impact over the last decade.

So what does a solution to the word count problem look like in bash-reduce? Let's
start with the mapper:

### mapper
{% highlight bash %}
{
  for(i = 1; i <= NF; i++) {
    gsub("[^A-Za-z0-9]", "", $i)
    print tolower($i), 1
  }
}
{% endhighlight %}
<br>

What does this do? awk by default splits an input line by whitespace and stores
each word in a variable named after the index of the word. The number of fields
is stored in the variable `NF`. So in the for-loop we are simply iterating over
all the words. `gsub` is a function that will replace anything matched by the
regular expression in the first argument with whatever is in the second argument,
in the string that is passed in the third. So we are just removing any non
alphanumeric content in the string. Then we return the word (in lower case) and the
number 1. The significance of this is that the word occurred once.

### reducer
{% highlight bash %}
{
  for(i = 2; i <= NF; i++) {
    sum += $i
  }
  print $1, sum
  sum = 0
}
{% endhighlight %}
<br>

Here we have the reduce function. The contract of the framework specifies that
in the input to the reduce function, the first field will be the key, and all the
subsequent fields will be the values for that key. So what this does is to iterate
over all the values and sum them up.

### let's try it
I downloaded the complete works of Shakespeare to use as input data when working
on this. I've found it to be small enough that any program will finish in a few
seconds or less, but complex enough to be mildly interesting. Since the framework
will sort the result by the second column by default, all we have to do to get
the top 10 words by occurrence in the complete works of Shakespeare is:

{% highlight bash %}
$ ./bash-reduce mappers/word-count.awk reducers/sum.awk data/shakespeare | head
the 27825
and 26791
i 20681
to 19261
of 18289
a 14667
you 13716
my 12481
that 11135
in 11027
{% endhighlight %}
<br>

bash-reduce lives [here](https://github.com/sorhus/bash-reduce).
