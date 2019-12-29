# First Steps with DataFrames in Julia

Back in the Summer of 2012 when my knowledge of programming was limited to
a couple of classes of C/C++ and R for maths undergraduates, I came across a
post on [R Bloggers](https://www.r-bloggers.com) about this interesting new
thing called "Julia" that looked a bit like R (to me) but was meant to
approach C levels of performance for numerical methods.

So I downloaded the Windows release, fired up the console, and checked that
`2 + 2` did indeed evaluate to `4`. I think I then tried to print 1 to 10 in a
for-loop, but I didn't immediately get the syntax right so I closed it down
and didn't touch Julia again for a good seven years.

A couple of months ago I decided to give it another shot, because it's just
gotten more and more interesting since then.

Unfortunately it has taken me a very long time to get off the ground.

Partially it's because I've been bumbling through tutorials and examples on
Github (after work, tired, with Netflix on in the same room)
that either assume too much knowledge of the language or dive really deeply
into the fundamentals of the language with a focus on what's important to
know for numerical methods applications.

Partially too it's because there seem to be several options and no clear-cut
favourite for much what I want to do.

I am a ~~statistician~~ data scientist. I need to
 * read data from a common file format like CSV or Parquet
 * inspect it in something like a data frame
 * write functions to do string manipulation
 * write functions to do numerical computation
 * apply functions to columns of a data frame (normalise a column, say, or filter out rows where `x < y`)
 * fit a statistical or machine learning model to a dataset
 * perform statistical inference (likelihood ratio tests and all that)
 * save and load a fitted model
 * use this fitted model to generate predictions on unseen data

I don't know whether to use `CSV.jl` or `CSVFiles.jl`, `Query.jl` or
 `DataFramesMeta.jl`,
`Statistics.jl` or `StatsBase.jl`, `MLDataUtils.jl`, `MLBase.jl` or the data
preprocessing functions from `Flux.jl`, the `|>` pipe operator, the `@linq`
macro or `Lazy.jl` and so on and so forth.

There are the JuliaML, JuliaData and JuliaStats organisations on Github.
There is also the QueryVerse, FluxML, OnlineStats and JuliaComputing.

I'm hoping that within weeks or months I'll want to edit this post because it
all becomes clear to me. Unfortunately it's all a bit of a muddle to me as of
the end of 2019.

Coming from the world of R, a lot of this is hidden from view.
You can read a csv, do a bit of data wranging, fit a (generalised) linear
model and do a bit of statisical inference without having to explicity
install or import a single package.
Of course in R a lot of the underlying stuff is indeed held in
different packages, and has been installed and imported on your behalf
at some point. Nevertheless, the narrower use case of R makes it much
easier for a beginner to actually _begin_.

The situation in Python is a bit more stable too - Pandas for data I/O and
general data manipulation, sklearn for very many different types of machine
learning model, and then something like XGBoost, PyTorch, Tensorflow or
statsmodels if you know you want to build a particular kind of model. That's a
simplification obviously, but not a huge one.

Anyway, let's get started with Julia. In this post I will import the Iris
dataset, which contains one string/categorical and four numeric columns.
I will do some manipulation of the string column, and I will filter out some rows
based on the value of one of the numeric columns.

## Getting Started

Open a Julia console
```
leo:~$ julia
               _
   _       _ _(_)_     |  Documentation: https://docs.julialang.org
  (_)     | (_) (_)    |
   _ _   _| |_  __ _   |  Type "?" for help, "]?" for Pkg help.
  | | | | | | |/ _` |  |
  | | |_| | | | (_| |  |  Version 1.2.0 (2019-08-20)
 _/ |\__'_|_|_|\__'_|  |  Official https://julialang.org/ release
|__/                   |

julia>
```

To install a given package, I opened a Julia console, press `]`, and then
typed `add {package name}`. Eg:
```julia
julia> ]
(v1.2) pkg> add RDatasets
(v1.2) pkg> (backspace)
julia>
```

Load the packages that we're going to use.
```julia
using CSV, DataFrames, DataFramesMeta, Lazy, GLM
```
Next, load the data from CSV into a DataFrame. We use `|>` to pipe the CSV
from the CSV format into a DataFrame. We also put a semicolon at the end
of the expression to suppress output to the console.
```julia
df = CSV.File("/path/to/iris.csv") |> DataFrame;
```
Alternatively, you can load the data directly from the RDatasets package.
```
using RDatasets
df = dataset("datasets", "iris");
```
Have a look at the first five rows of the DataFrame. In R, this function is
called `head`, while in Python it's `df.head(5)`.
```julia
first(df, 5)

5×5 DataFrame
│ Row │ SepalLength │ SepalWidth │ PetalLength │ PetalWidth │ Species      │
│     │ Float64     │ Float64    │ Float64     │ Float64    │ Categorical… │
├─────┼─────────────┼────────────┼─────────────┼────────────┼──────────────┤
│ 1   │ 5.1         │ 3.5        │ 1.4         │ 0.2        │ setosa       │
│ 2   │ 4.9         │ 3.0        │ 1.4         │ 0.2        │ setosa       │
│ 3   │ 4.7         │ 3.2        │ 1.3         │ 0.2        │ setosa       │
│ 4   │ 4.6         │ 3.1        │ 1.5         │ 0.2        │ setosa       │
│ 5   │ 5.0         │ 3.6        │ 1.4         │ 0.2        │ setosa       │

```
Let's also look at the distinct values of the `Species` column
```julia
unique(df.Species)

3-element Array{String,1}:
 "setosa"    
 "versicolor"
 "virginica"
```

## Very Simple Data Transformations

Now we'll define a function to create a binary variable from the `Species`
column - 1 if the species is "setosa", and 0 otherwise.

I'm sure this could be done using either inbuilt `DataFrames` methods or the
`@transform` macro from `DataFramesMeta.jl`, but I would like to give an
example of defining a (very simple) function in Julia too.

The `::Int32` specifies that we want the function to return a 32-bit integer.
It's not strictly necessary here, but it's nice to have that level of control.

```julia
function is_setosa(x)::Int32
	return 1 * (x == "setosa")
end

is_setosa (generic function with 1 method)
```
In a moment I will do several steps of manipulation in the one go using
`Lazy.jl`, but for now I'll throw caution to the wind and keep overwriting the
`df` variable with updated versions of the DataFrame.

We'll add a new column to the DataFrame using this function. We need to add a
full stop after the function name to make it work on every row of the
DataFrame. This is one of those times where I'm not sure if what's I'm doing
is good practice. Coming from R and Python, vectorisation of functions often
"just works", with little or no difference between a function designed to work
on a string or an array of strings.

We'll also filter out all the rows with `SepalLength` of greater than or equal
to 7.0, just because.

```julia
df = @transform(df, setosa = is_setosa.(:Species));

# note ".<" and not just "<", because it's applied to every row
df = @where(df, :SepalLength .< 7.0);
```
Let's quickly inspect the DataFrame again to be sure we did what we wanted
```julia
first(df, 5)


5×6 DataFrame
│ Row │ SepalLength │ SepalWidth │ PetalLength │ PetalWidth │ Species      │ setosa │
│     │ Float64     │ Float64    │ Float64     │ Float64    │ Categorical… │ Int32  │
├─────┼─────────────┼────────────┼─────────────┼────────────┼──────────────┼────────┤
│ 1   │ 5.1         │ 3.5        │ 1.4         │ 0.2        │ setosa       │ 1      │
│ 2   │ 4.9         │ 3.0        │ 1.4         │ 0.2        │ setosa       │ 1      │
│ 3   │ 4.7         │ 3.2        │ 1.3         │ 0.2        │ setosa       │ 1      │
│ 4   │ 4.6         │ 3.1        │ 1.5         │ 0.2        │ setosa       │ 1      │
│ 5   │ 5.0         │ 3.6        │ 1.4         │ 0.2        │ setosa       │ 1      │

```

It's less straightforward than I would like to select _everything except_ a
given column. Below is a two-line way of doing it. Again, maybe it's just my
inexperience showing.
```julia
cols = filter(a -> a != :Species, names(df))
X = @select(df, cols)

# what exactly have we done here?
first(X, 5)

5×5 DataFrame
│ Row │ SepalLength │ SepalWidth │ PetalLength │ PetalWidth │ setosa │
│     │ Float64     │ Float64    │ Float64     │ Float64    │ Int32  │
├─────┼─────────────┼────────────┼─────────────┼────────────┼────────┤
│ 1   │ 5.1         │ 3.5        │ 1.4         │ 0.2        │ 1      │
│ 2   │ 4.9         │ 3.0        │ 1.4         │ 0.2        │ 1      │
│ 3   │ 4.7         │ 3.2        │ 1.3         │ 0.2        │ 1      │
│ 4   │ 4.6         │ 3.1        │ 1.5         │ 0.2        │ 1      │
│ 5   │ 5.0         │ 3.6        │ 1.4         │ 0.2        │ 1      │

```

Now let's chain all this together using the `@>` macro from `Lazy.jl`. This
looks a lot like operations on Spark DataFrames to me. I believe it's also
quite similar to LINQ in C#.

```julia
X = @> begin
	df
	@transform(y = is_setosa.(:Species))
	@where(:SepalLength .< 7.0)
	@select(cols)
end

# Inspect our work
first(X, 5)

5×5 DataFrame
│ Row │ SepalLength │ SepalWidth │ PetalLength │ PetalWidth │ setosa │
│     │ Float64     │ Float64    │ Float64     │ Float64    │ Int32  │
├─────┼─────────────┼────────────┼─────────────┼────────────┼────────┤
│ 1   │ 5.1         │ 3.5        │ 1.4         │ 0.2        │ 1      │
│ 2   │ 4.9         │ 3.0        │ 1.4         │ 0.2        │ 1      │
│ 3   │ 4.7         │ 3.2        │ 1.3         │ 0.2        │ 1      │
│ 4   │ 4.6         │ 3.1        │ 1.5         │ 0.2        │ 1      │
│ 5   │ 5.0         │ 3.6        │ 1.4         │ 0.2        │ 1      │

```

## Fitting Linear Models

We can fit linear and generalised linear models to the data in the DataFrame
using the `GLM.jl` package.

I haven't done any train-test split or data normalisation here. I'll leave
that for a later post.
```julia
lm1 = lm(@formula(SepalLength ~ SepalWidth + PetalLength + PetalWidth + setosa), X)

StatsModels.TableRegressionModel{LinearModel{GLM.LmResp{Array{Float64,1}},GLM.DensePredChol{Float64,LinearAlgebra.Cholesky{Float64,Array{Float64,2}}}},Array{Float64,2}}

SepalLength ~ 1 + SepalWidth + PetalLength + PetalWidth + setosa

Coefficients:
───────────────────────────────────────────────────────────────────────────────
               Estimate  Std. Error    t value  Pr(>|t|)  Lower 95%   Upper 95%
───────────────────────────────────────────────────────────────────────────────
(Intercept)   2.30873     0.306429    7.53429     <1e-11   1.70258    2.91488  
SepalWidth    0.585309    0.0864089   6.7737      <1e-9    0.414383   0.756234 
PetalLength   0.565343    0.0873592   6.47148     <1e-8    0.392538   0.738148 
PetalWidth   -0.336146    0.144578   -2.32501     0.0216  -0.622136  -0.0501558
setosa       -0.0530058   0.200776   -0.264005    0.7922  -0.45016    0.344148 
───────────────────────────────────────────────────────────────────────────────



```

Note that fitting a generalised linear model with Gaussian errors and
identity link is the same thing as fitting an Ordinary Least Squares
linear model.

```julia
lm2 = glm(@formula(SepalLength ~ SepalWidth + PetalLength + PetalWidth + setosa), X, Normal(), IdentityLink())


tatsModels.TableRegressionModel{GeneralizedLinearModel{GLM.GlmResp{Array{Float64,1},Normal{Float64},IdentityLink},GLM.DensePredChol{Float64,LinearAlgebra.Cholesky{Float64,Array{Float64,2}}}},Array{Float64,2}}

SepalLength ~ 1 + SepalWidth + PetalLength + PetalWidth + setosa

Coefficients:
───────────────────────────────────────────────────────────────────────────────
               Estimate  Std. Error    z value  Pr(>|z|)  Lower 95%   Upper 95%
───────────────────────────────────────────────────────────────────────────────
(Intercept)   2.30873     0.306429    7.53429     <1e-13   1.70814    2.90932  
SepalWidth    0.585309    0.0864089   6.7737      <1e-10   0.41595    0.754667 
PetalLength   0.565343    0.0873592   6.47148     <1e-10   0.394122   0.736564 
PetalWidth   -0.336146    0.144578   -2.32501     0.0201  -0.619514  -0.0527777
setosa       -0.0530058   0.200776   -0.264005    0.7918  -0.446519   0.340507 
───────────────────────────────────────────────────────────────────────────────
```

Finally, a generalised linear model where the response is "Setosa" or
"Not Setosa". Note that this will only work if response is Integer type.
Having a Float32 response failed on me.

```julia
glm1 = glm(@formula(setosa ~ SepalLength + SepalWidth + PetalLength + PetalWidth), X, Binomial(), LogitLink())

StatsModels.TableRegressionModel{GeneralizedLinearModel{GLM.GlmResp{Array{Float64,1},Binomial{Float64},LogitLink},GLM.DensePredChol{Float64,LinearAlgebra.Cholesky{Float64,Array{Float64,2}}}},Array{Float64,2}}

setosa ~ 1 + SepalLength + SepalWidth + PetalLength + PetalWidth

Coefficients:
────────────────────────────────────────────────────────────────────────────────
              Estimate  Std. Error       z value  Pr(>|z|)  Lower 95%  Upper 95%
────────────────────────────────────────────────────────────────────────────────
(Intercept)   -6.99364    37161.7   -0.000188195    0.9998   -72842.5    72828.5
SepalLength    7.05419    11558.7    0.000610293    0.9995   -22647.6    22661.7
SepalWidth     6.94495     5672.01   0.00122443     0.9990   -11110.0    11123.9
PetalLength  -14.4666      9288.48  -0.00155748     0.9988   -18219.6    18190.6
PetalWidth   -17.6012     13844.2   -0.00127138     0.9990   -27151.7    27116.5
────────────────────────────────────────────────────────────────────────────────
```

The outputs of GLM are similar to those of R or StatsModels in Python - with a nicely
printed table of coefficient estimates, standard errors and confidence intervals.

