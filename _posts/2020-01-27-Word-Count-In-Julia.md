# Simple Word Count in Julia

## Using only one external package!

The word count program is a sort of "Hello World" of distributed computing,
[from basic Hadoop](https://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html#Example:_WordCount_v1.0)
[to the newest versions of Spark](https://spark.apache.org/examples.html)
and everything in between.

The aim of the program is to read a (potentially very large) text input file line-by-line,
split each line into individual words, and then keep count of the number of occurrences
of each word, returning a collection of `(word, count)` pairs.

This was a popular exercise for learning Hadoop MapReduce because it required you to write
both a mapper and a reducer, neither of which needed to be particularly complex,
and was also simple enough that you could test the program in a normal
Unix terminal using `cat` and pipes.

For similar reasons it is also a good place to get started with the RDD API in Spark.

Seeing as how I spend a large chunk of my working day working with PySpark and HDFS
I figured I would try it myself in Julia.

I'm not going to try distributing anything - I'll just count the words in a single string.
I'll also try to do it in a fairly functional style using Mike Innes'
[Lazy.jl](https://github.com/MikeInnes/Lazy.jl)

I didn't really want to have to use any external package at all,
but I found that the inbuilt pipe operator `|>` doesn't handle functions with multiple arguments
as cleanly as I would have hoped.

I was able to get what I wanted with the `@as` macro `Lazy.jl`, even though it took me a while
to get going.

All the examples in the documentation use an underscore as the placeholder
argument for the functions in the macro, but this appears to be no longer allowed
in Julia 1.2.0.

The code below will fail with the error `ERROR: all-underscore identifier used as rvalue`.

```julia
using Lazy

sentence = "a way a lone a last a loved a long the riverrun"

@as _ sentence begin
    split(_, " ")
    map(a -> (a, 1), _)
end
```
However, simply avoiding the underscore seems to be ok:
```julia
using Lazy

sentence = "a way a lone a last a loved a long the riverrun"

@as x sentence begin
    split(x, " ")
    map(a -> (a, 1), x)
end
```

Another problem is that `map` doesn't work on Julia dictionaries. This was a complication because
the Scala one-liner word count created a Map (a dictionary in Julia or Python) of (word, list of word occurrances) key-value pairs
and then applied a `mapValues` method that got the length of each value (the word count, essentially).

One solution would have been to use `@as` to build up the dictionary, and then iterate through it using `pairs`:

```julia
d = @as x sentence begin
    split(x, " ")
    map(a -> (a, 1), x)
    groupby(a -> a[1], x)
end

for x in pairs(d)
    println(x[1], ": ", length(x[2]))
end
```
However this felt a bit ugly, and broke the functional, piping look I was going for.

Luckily, for reasons I do not understand, `Lazy.jl`'s `lazymap` worked just fine on the dictionary where `map` did not.


## Word Count

```julia
using Lazy

sentence = "a way a lone a last a loved along the riverrun"

# NB: You can't use _ as the placeholder, even
# though it's still in all the examples on Github
@as x sentence begin
    split(x, " ")
    map(a -> (a, 1), x)
    groupby(a -> a[1], x)
    lazymap(a -> (a[1], length(a[2])), x)  # lazymap works here but map does not
end
```

## Bonus Scala Word Count

Below is how it can be implemented in Scala. It's fairly similar to the Spark
RDD API. The biggest difference is that non-distributed Scala has no `reduceByKey` that I am aware of.

```scala
val sentence = "a way a lone a last a loved along the riverrun"

val counts = sentence.split(" ").map(a => (a, 1)).groupBy(_._1).mapValues(_.size)

println(counts)
```

