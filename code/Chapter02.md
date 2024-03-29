---
layout: code
title: Chapter02
permalink: /code/Chapter02.html
---
## Chapter 2

### Timing Julia
`@time` is the simplest way to measure execution times

```julia
@time sqrt.(rand(1000));
```

`@time` can also be used for any expression

```julia
s=0
@time for i=1:1000
   global s=s+sqrt(i)
end
```

`@timev` provides additoinal information

```julia
@timev sqrt.(rand(1000));
```

`@time` returns the value of the expression

```julia
sum(@time sqrt.(rand(1000)))
```

`@elapsed` returns the elapsed execution time as a value ...

```julia
@elapsed sqrt.(rand(1000))
```

... and hence can be used in automated tests

```julia
using Test
@test @elapsed(sqrt.(rand(1000))) <= 10e-4
```

### Profiling

Load the `Profile` stdlib module

```julia
using Profile
```

We profile a simple function that creates a random matrix, and then
computes the mean of the square of each row.

```julia
using Statistics
function randmsq()
    x = rand(10000, 1000)
    y = mean(x.^2, dims = 1)
    return y
end
```

Run the function once before profiling, to compile it

```julia
randmsq()
```

Execute the function while profiling it

```julia
@profile randmsq()
```

`print` displays a tree view of the execution traces

```julia
Profile.print()
```

Clear saved profiles

```julia
Profile.clear()
```

Use the `init` function to configure the profiler

```julia
Profile.init(delay=.01)
```

Run the profile using new setting

```julia
@profile randmsq()
```

Profileview can be used to visualise the
`] add ProfileView` to add the package

```julia
using Pkg
Pkg.add("ProfileView")

using ProfileView
ProfileView.view()
ProfileView.svgwrite("profile_results.svg")
```

### Statistically significant benchmarking
`]add BenchmarkTools` to add the package

```julia
Pkg.add("BenchmarkTools")
using BenchmarkTools
@benchmark sqrt.(rand(1000))

@btime mean(rand(1000));
```

