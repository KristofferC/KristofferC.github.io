+++
date = "2016-12-26T16:00:00"
draft = true
tags = ["plot", "adaptivity"]
title = "Adaptively refine a grid to smoothly plot functions"
math = true
summary = """
We make an adaptive grid generator that computes the points to evaluate the function such that the result is a nice looking smooth plot.
"""
+++

A recent issue on the [Plots.jl](https://github.com/tbreloff/Plots.jl/issues/621) repo seemed like it contained a fun little side project to implement.
The problem is to create a grid for a user provided function on a given interval such that plotting the function at those grid points results in a smooth aesthetic plot. At the same time, the function could be expensive to evaluate so we would like to reduce the number of function evaluations.








## Avoiding aliasing





### Future work


Dealing with y-lims.

{{< term >}}
<span class=jl_prompt>julia> </span><span class=jl_input>I write stuff </span>
{{< /term >}}

dasds