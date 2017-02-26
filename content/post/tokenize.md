+++
date = "2017-02-26T15:00:00"
draft = false
tags = ["lexing", "tokenization", "julia"]
title = "What identifier is the most common in Julia code? The answer might surprise you!"
math = true
summary = """
The julia package *Tokenize* is used to perform lexical analysis on Julia source code
and the number of occurrences of identifiers is investigated for Julia 0.5 and 0.6 as well as a large corpus of source code from packages.
What identifier will win? Read to find out!
"""
+++

[*Tokenize*](https://github.com/KristofferC/Tokenize.jl) is a Julia package to perform lexical analysis of Julia source code.
Lexing is the process of transforming raw source code (represented as normal text) into a sequence of *tokens* which is
a string with an associated meaning. "Meaning" could here be if the string represent an operator, a keyword, a comment etc.

The example below shows lexing (or tokenization) of some simple code.

```julia
julia> using Tokenize

julia> collect(tokenize("""
       100
       "this is a string"
       'a'
       type
       foo
       *
       """))
13-element Array{Tokenize.Tokens.Token,1}:
 1,1-1,3          INTEGER        "100"
 1,4-2,0          WHITESPACE     "\n"
 2,1-2,18         STRING         "\"this is a string\""
 2,19-3,0         WHITESPACE     "\n"
 3,1-3,3          CHAR           "'a'"
 3,4-4,0          WHITESPACE     "\n"
 4,1-4,4          KEYWORD        "type"
 4,5-5,0          WHITESPACE     "\n"
 5,1-5,3          IDENTIFIER     "foo"
 5,4-6,0          WHITESPACE     "\n"
 6,1-6,1          OP             "*"
 6,2-7,0          WHITESPACE     "\n"
 7,1-7,0          ENDMARKER      ""
```

The displayed array containing the tokens has three columns. The first column shows the location where the string of the token starts and ends,
which is represented as the line number (row) and at how many characters into the line (columns) the token starts / ends.
The second column shows the type (*kind*) of token and, finally, the right column shows the string the token contains.

One of the different token kinds is the *identifier*. These are names that refer to different entities in the code.
This includes variables, types, functions etc. The name of the identifiers are chosen by the programmer,
in contrast to keywords which are chosen by the developers of the language.
Some questions I thought interesting are:

* What is the most common identifier in the Julia Base code (the code making up the standard library). Has it changed from 0.5 to 0.6?
* How about packages? Is the source code there significantly different from the code in Julia Base in terms of the identifiers used?

The plan is to use *Tokenize* to lex both Julia Base and a bunch of packages, count the number of occurrences of
each identifier and then summarize this as a top 10 list.

## A Julia source code identifier counter

First, let's create a simple counter type to keep track of how many times each identifier occur.
This is a just a wrapper around a dictionary with a default value of `0` and a
`count!` method that increments the counter for the supplied key:

```julia
immutable Counter{T}
    d::Dict{T, Int}
end
Counter{T}(::Type{T})= Counter(Dict{T, Int}())

Base.getindex{T}(c::Counter{T}, v::T) = get(c.d, v, 0)
getdictionary(c::Counter) = c.d
count!{T}(c::Counter{T}, v::T) = haskey(c.d, v) ? c.d[v] += 1 : c.d[v] = 1
```

A short example of the `Counter` type in action is showed below.

```julia
julia> c = Counter(String)
Counter{String}(Dict{String,Int64}())

julia> c["foo"]
0

julia> count!(c, "foo"); count!(c, "foo");

julia> c["foo"]
2
```

Now, we need a function that tokenizes a file and counts the number of identifiers in it.
The code for such a function is shown below and a short explanation follows:

```julia
function count_tokentypes!(counter, filepath, tokentype)
    f = open(filepath, "r")
    for token in tokenize(f)
        if Tokens.kind(token) == tokentype
            count!(counter, untokenize(token))
        end
    end
    return counter
end
```

This opens the file at the path `filepath`, loops over the tokens, and if the kind of token is the `tokentype`
the `counter` is incremented with the string of the token (extracted with `untokenize`) as the key.
In *Tokenize* each type of token is represented by an enum, and the one corresponding to identifiers is named
`Tokens.IDENTIFIER`.

As an example, we could run the function on a short file in base (`nofloat_hashing.jl`):

```julia
julia> BASEDIR =  joinpath(JULIA_HOME, Base.DATAROOTDIR, "julia", "base")

julia> filepath = joinpath(BASEDIR, "nofloat_hashing.jl");

julia> c = Counter(String);

julia> count_tokentypes!(c, filepath, Tokens.IDENTIFIER)
Counter{String}(Dict("b"=>2,"x"=>8,"a"=>2,"h"=>8,"UInt32"=>1,"UInt16"=>1,"hx"=>3,"abs"=>1,"Int8"=>1,"Int16"=>1…))

julia> c["h"]
8
```

We see here that there are 8 occurrences of the identifier `h` in the file.

The next step is to apply the `count_tokentypes` function to *all* the files in the base directory.
To that end, we create the `applytofolder` function:

```julia
function applytofolder(path, f)
    for (root, dirs, files) in walkdir(path)
        for file in files
            f(joinpath(root, file))
        end
    end
end
```

It takes a `path` to a folder and applies the function `f` on each file in that path.
The `walkdir` function works recursively so each file will be visited this way.

Finally, we create a `Counter` and call the previously created `count_tokentypes` on all files
that end with `".jl"` using the `applytofolder` function:

```julia
julia> BASEDIR = joinpath(JULIA_HOME, Base.DATAROOTDIR, "julia", "base")

julia> c = Counter(String)

julia> applytofolder(BASEDIR,
                     function(file)
                         if endswith(file, ".jl")
                             count_tokentypes!(c, file, Tokens.IDENTIFIER)
                         end
                     end)
```

The counter `c` now contains the count of all identifiers in the base folder:

```julia
julia> c["_uv_hook_close"]
12

julia> c["x"]
7643

julia> c["str"]
230
```

## Analysis

We are interested in the most common identifiers so we create a function that
extracts the `n` most common identifiers as two vectors.
One with the identifiers and one with the counts:

```julia
function getntop(c::Counter, n)
    vec = Tuple{String, Int}[]
    for (k, v) in getdictionary(c)
        push!(vec, (k, v))
    end
    sort!(vec, by = x -> x[2], rev = true)
    vec_trunc = vec[1:n-1]
    identifiers = [v[1] for v in vec_trunc]
    counts      = [v[2] for v in vec_trunc]
    return identifiers, counts
end
```

To visualize this we use the excellent plotting package
 [*UnicodePlots*](https://github.com/Evizero/UnicodePlots.jl):

```julia
julia> using UnicodePlots

julia> identifiers, counts = getntop(c, 10)

julia> barplot(identifiers, counts, title = "Base identifiers")
                   Base identifiers
       ┌────────────────────────────────────────┐
     x │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 7643 │
     T │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 7202   │
     A │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 7001    │
     i │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 5119            │
   Ptr │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 4239                │
     s │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 4128                 │
     n │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 3650                   │
     B │▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 3143                     │
    io │▪▪▪▪▪▪▪▪▪▪▪▪ 2714                       │
       └────────────────────────────────────────┘
```

So there we have it, `x` is the winner, closely followed by `T` and `A`.
This is perhaps not very surprising; `x` is a very common variable name,
`T` is used a lot in parametric functions and `A` is used a lot in the
Linear Algebra code base which is quite large.

### Difference vs. 0.6

The plot below shows the same experiment repeated on the 0.6 code base:


```julia
               Base identifiers 0.6
       ┌────────────────────────────────────────┐ 
     x │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 7718 │ 
     A │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 7313   │ 
     T │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 6932    │ 
     i │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 5242            │ 
   Ptr │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 4147                 │ 
     s │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 4093                 │ 
     n │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 3650                   │ 
     B │▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 3174                     │ 
    io │▪▪▪▪▪▪▪▪▪▪▪▪▪ 2933                      │ 
       └────────────────────────────────────────┘ 
```

Most of the counts are relatively similar between 0.5 and 0.6 with the exception that `A` overtook `T` for the second place.
In fact, the number of `T` identifers have decreased with almost 300 counts!
What could have caused this?
The answer is a new syntactic sugar feature available in Julia 0.6 which was implemented by Steven G. Johnson in [PR #20414](https://github.com/JuliaLang/julia/pull/20414) .
This allowed a parametric function with the syntax

```jl
foo{T <: Real}(Point{T}) = ...
```

to instead be written more tersely as

```
foo(Point{<:Real})...
```

In [PR #20446](https://github.com/JuliaLang/julia/pull/20446) Pablo Zubieta went through the Julia code base and updated
many of the function signatures to use this new syntax.
Since `T` is a very common name to use for the parameter, the counts of `T` significantly decreased.
And this is how `A` managed to win over `T` in 0.6 in the prestigeful "most common identifier"-competition.

### Julia packages.

We now perform the same experiment but on the Julia package directory. For me, this includes around 130 packages:

```julia
julia> length(readdir(Pkg.dir()))
137
```

The results are:

```julia
                   Package identifiers
        ┌────────────────────────────────────────┐ 
      T │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 15425 │ 
      x │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 15062  │ 
   test │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 13624     │ 
      i │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 9989              │ 
      d │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 9562               │ 
      A │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 8280                 │ 
    RGB │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 8041                  │ 
      a │▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 7144                    │ 
      n │▪▪▪▪▪▪▪▪▪▪▪▪▪▪ 6470                     │ 
        └────────────────────────────────────────┘ 
```

When we counted the Julia base folder we excluded all the files used for unit testing.
For packages, these files are included and clearly `test`, used in the `@test` macro, is
unsurprisingly very common. `T`, `x` and `i` are common in packages and Base but for some reason
the variable `d` is more common in packages than in Base.

## Conclusion

Doing these type of investigations has perhaps little practical use but it is, at leat to me, a lot of fun.
Feel free to tweak the code to find the most common string literal (`Tokens.STRING`) or perhaps most common integer (``Tokens.INTEGER`)
or anything else you can come up with.

Below is a wordcloud I made with the top 50 identifiers in Julia Base.

{{< figure src="/img/wordcloud.png" >}}
