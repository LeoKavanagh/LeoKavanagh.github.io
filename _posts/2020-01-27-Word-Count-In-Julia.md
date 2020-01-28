## Using only one external package!

The word count program is a sort of "Hello World" of distributed computing,
[from basic Hadoop](https://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html#Example:_WordCount_v1.0)
to the [newest versions of Spark](https://spark.apache.org/examples.html)
and everything in between.

Seeing as how I spend a large chunk of my working day working with PySpark and HDFS
I figured I would try it myself in Julia.

The aim of the program is to read a (potentially very large) text input file line-by-line,
split each line into individual words, and then keep count of the number of occurrences
of each word (without running out of memory), returning a collection of `(word::String, count::Int)` pairs.

This was/is a popular exercise for learning Hadoop MapReduce because it requires
both a mapper and a reducer, neither of which need to be particularly complex.
It tends to be possible to test the program in a normal
Unix terminal using `cat` and pipes, depending on how you've written it. For similar reasons it is also a good place to get started with the RDD API in Spark.


The basics of a map-reduce style word count program is as follows:

### Map Step
For each line in the input file:
 * split the line into words (we'll just split on whitespace)
 * create a `(word, 1)` tuple out of each word
 * emit each `(word, 1)` tuple in turn

If you're running on Hadoop, the outputs of the mapper will be shuffled and sorted,
and then sent to the reducer.

### Reduce Step
Collect each tuple outputted by the mapper:
 * if the word has not been seen yet, create a `(word: 1)` key-value pair
 * if the word has been seen already, increment the count value of its key: `(word: 2)`

Finally, print the key-value pairs of word counts created by the reducer.

## Julia Implementation

I'm not going to try distributing anything - I'll just count the words in a single string, in a single Julia console.

The Map step from above is reproduced pretty much
verbatim, only instead of printing each tuple we just create an array of them and hold
it in memory.

So for example, the string
```julia
"a way a lone a last a loved a long the riverrun"
```
will become 

```julia
12-element Array{Tuple{SubString{String},Int64},1}:
 ("a", 1)       
 ("way", 1)     
 ("a", 1)       
 ("lone", 1)    
 ("a", 1)       
 ("last", 1)    
 ("a", 1)       
 ("loved", 1)   
 ("a", 1)       
 ("long", 1)    
 ("the", 1)     
 ("riverrun", 1)
```

The Reduce step is slightly different, because we aren't piping all these
`(word, 1)` tuples through a reducer one at a time. 
We do a SQL-style `GROUP BY` on the first element of each tuple (all the words)
in the array to create a `dictionary`.
The key is the word itself, and the value is all its `(word, 1)` tuples:

```julia
Dict{Any,Any} with 8 entries:
  "loved"    => Any[("loved", 1)]
  "long"     => Any[("long", 1)]
  "the"      => Any[("the", 1)]
  "lone"     => Any[("lone", 1)]
  "last"     => Any[("last", 1)]
  "riverrun" => Any[("riverrun", 1)]
  "a"        => Any[("a", 1), ("a", 1), ("a", 1), ("a", 1), ("a", 1)]
  "way"      => Any[("way", 1)]

```

Finally, we just calculate the length of each entry in the dictionary.

I'm going to use Mike Innes'
[Lazy.jl](https://github.com/MikeInnes/Lazy.jl)
package for this, because I like trying to add a little bit of functional programming to
what I'm doing, and because I found that the inbuilt pipe operator `|>`
doesn't handle functions with multiple arguments as cleanly as I would have hoped.

I was able to get what I wanted with the `@as` macro from `Lazy.jl`,
even though it took me a while to get going.

All the examples in the documentation use an underscore as the placeholder
argument for the functions in the macro, but this appears to be no longer allowed
in Julia 1.2.0.

The code below shows the first couple of steps of the word count program:
split the string on whitespace, and then
map each word to a `(word, 1)` tuple (for Hadoopish reasons).

However, it will fail with the error `ERROR: all-underscore identifier used as rvalue`.

```julia
using Lazy

sentence = "a way a lone a last a loved a long the riverrun"

@as _ sentence begin
    split(_, " ")
    map(a -> (a, 1), _)
end
```

Simply avoiding the underscore seems to be ok:
```julia
using Lazy

sentence = "a way a lone a last a loved a long the riverrun"

@as x sentence begin
    split(x, " ")
    map(a -> (a, 1), x)
end
```

Another problem is that `map` doesn't work on Julia dictionaries. This was a complication because
the Scala one-liner word count I used as inspiration (see below)
created a Map (a dictionary in Julia or Python) of `(word, list of word occurrances)` key-value pairs
and then applied a `mapValues` method that got the length of each value (the word count, essentially).

One Julia solution would have been to use `@as` to build up the dictionary, and then iterate through it using `pairs`:

```julia
using Lazy

d = @as x sentence begin
    split(x, " ")
    map(a -> (a, 1), x)
    groupby(a -> a[1], x)
end

for x in pairs(d)
    println(x[1], ": ", length(x[2]))
end
```
However this felt a bit ugly, and broke the functional, piping style I was going for.

Luckily, for reasons I do not understand, `lazymap` from `Lazy.jl` worked just fine
on the dictionary where `map` did not.

## Word Count

```julia
using Lazy

sentence = "a way a lone a last a loved a long the riverrun"

# NB: You can't use _ as the placeholder, even
# though it's still in the examples on Github
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
val sentence = "a way a lone a last a loved a long the riverrun"

val counts = sentence.split(" ").map(a => (a, 1)).groupBy(_._1).mapValues(_.size)

println(counts)
```

