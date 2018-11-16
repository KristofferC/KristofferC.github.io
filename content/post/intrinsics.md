+++
date = "2018-11-13T15:00:00"
draft = false
tags = ["simd", "intrinsics", "julia"]
title = "SIMD and SIMD-intrinsics in Julia"
math = true
summary = """
Short guide on SIMD and how to call (SIMD) intrinsics in the Julia programming language.
"""
+++

## Introduction

The good old days of processors getting a higher clock-speed every year have been over for quite some time now.
Instead, other features of CPUs are getting improved like the number of cores, the size of the cache, and the instruction set
they support.
In order to be responsible programmers, we should try our best to take advantage of the features the hardware
provides.
One way of doing this is multithreading which is exploiting the fact that modern CPUs have multiple cores and
can run multiple (possibly independent) instruction streams simultaneously.
Another feature to take advantage of is that a single core on modern processors can do operations on **multiple** values in one instruction (called SIMD - Single Instruction Multiple Data).

This article is intended to give a short summary of using SIMD in the Julia programming language.
It is intended for people already quite familiar with Julia.
The first part is likely familiar to people that have been using Julia for a while, the latter part, which is about explicitly calling
SIMD intrinsics might be new. Feel free to scroll down to the intrinsic bit or read the TLDR about it below.

## TLDR (SIMD intrinsics)

To call an intrinsic like `_mm_aesdec_si128`:

* call the intrinsic from C
* use Clang with `-emit-llvm` to figure out the LLVM intrinsic ([for example](https://godbolt.org/z/vBTDVy))
* call the intrinsic from Julia with `ccall("insert_llvm_intrinsic", llvmcall, return_type, input_types, args...)`
  where an LLVM vector type like `<2 x i64>` is translated to the Julia type `NTuple{2, VecElement{Int64}}`

For example:

```jl
julia> const __m128i = NTuple{2, VecElement{Int64}};

julia> aesdec(a, roundkey) = ccall("llvm.x86.aesni.aesdec", llvmcall, __m128i, (__m128i, __m128i), a, roundkey);

julia> aesdec(__m128i((213132, 13131)), __m128i((31231, 43213)))
(VecElement{Int64}(-1627618977772868053), VecElement{Int64}(999044532936195731))
```

## Automatic SIMD

The backend compiler for Julia is LLVM which can in some cases vectorize loops using the [Loop Vectorizer](https://llvm.org/docs/Vectorizers.html#the-loop-vectorizer)
and it can even promote scalar code to SIMD operations using the [SLP Vectorizer](https://llvm.org/docs/Vectorizers.html#the-loop-vectorizer).

### Automatic loop vectorization

Defining a simple loop that does an "axpy" like operation `c .= a .* b`

```jl
function axpy!(c::Array, a::Array, b::Array)
    @assert length(a) == length(b) == length(c)
    @inbounds for i in 1:length(a)
        c[i] = a[i] * b[i]
    end
end
```

and using the code introspection tool to inspect the generated LLVM IR, we see:

```jl
julia> V64 = Vector{Float64}
Array{Float64,1}

julia> code_llvm(axpy!, Tuple{V64, V64, V64})
...
  %56 = fmul <4 x double> %wide.load, %wide.load24
  %57 = fmul <4 x double> %wide.load21, %wide.load25
  %58 = fmul <4 x double> %wide.load22, %wide.load26
  %59 = fmul <4 x double> %wide.load23, %wide.load27
...
```

The type `<4 x double>` is in LLVM IR terminology a [vector](https://llvm.org/docs/LangRef.html#vector-type), and is here the resulting type
of adding two arguments of the same vector type.
As a note, LLVM has also decided to unroll the loop by a factor of four.
Moving on to look at the assembly, we see that indeed (at least on the computer of the author), this LLVM IR has turned into processor instructions that
does four multiplications in one instruction.

```jl
julia> code_native(axpy!, Tuple{V64, V64, V64})
 vmulpd  (%esi,%ecx,8), %ymm0, %ymm0
 vmulpd  32(%esi,%ecx,8), %ymm1, %ymm1
 vmulpd  64(%esi,%ecx,8), %ymm2, %ymm2
 vmulpd  96(%esi,%ecx,8), %ymm3, %ymm3
```

The instruction [`vmulpd`](https://www.felixcloutier.com/x86/MULPD.html) does "packed" double-precision floating point addition and
the `ymm` registers fit 256 bits which can thus fit four 64-bit floats.

Any type of control flow inside the loop will likely mean the loop will not vectorize. That is why `@inbounds` is important here,
otherwise we have control flow to the part that throws the bounds error.

Note that for reductions using non-associative arithmetic (like floating point airthmetic) you will have to tell the compiler that it is ok to reorder
the accumulations into the reduction variable using the `@simd` macro.


### Automatic scalar vectorization

LLVM can also auto-vectorize scalar operations that follow a certain pattern. Here
is a function that does not contain any loop:

```jl
function mul_tuples(a::NTuple{4,Float64}, b::NTuple{4,Float64})
    return (a[1]*b[1], a[2]*b[2], a[3]*b[3], a[4]*b[4])
end
```

The function `mul_tuples` just multiplies numbers from two tuples of length four and forms a new tuple.
The pattern here should be obvious, it is clear that the four additions
could be done at the same time.
LLVM can identify such patterns and generate code that uses SIMD. Again,
inspecting the code we find that SIMD instructions are used:

LLVM IR:

```jl
julia> code_llvm(mul_tuples, Tuple{NTuple{4,Float64}, NTuple{4, Float64}})
...
  %7 = fmul <4 x double> %4, %6
...
```

Native code:

```jl
julia> code_native(mul_tuples, Tuple{NTuple{4,Float64}, NTuple{4, Float64}})
...
  vmulpd  (%edx), %ymm0, %ymm0
...
```

The scalar auto-vectorizer is quite impressive. It manages for example to very nicely vectorize a 4x4 matrix multiplication
in the [StaticArrays.jl package](https://github.com/JuliaArrays/StaticArrays.jl) which you can see doing something like

```jl
julia> using StaticArrays # import Pkg, Pkg.add("StaticArrays") to install

julia> @code_native rand(SMatrix{4,4}) * rand(SMatrix{4,4})
```

which almost only uses SIMD-instructions without StaticArrays.jl having to do any work for it.

# SIMD using  a vector library

While the auto-vectorizer can sometimes work pretty well, it quite easily gets confused. Alternatively, the data
is not laid out in such a way that it is allowed or beneficial to vectorize the code.
For example, trying a matrix multiplication of size
3x3 instead of 4x4 matrices in StaticArrays.jl and things are not so pretty anymore:

```jl
@code_native rand(SMatrix{3,3}) * rand(SMatrix{3,3})
    vmovsd  (%rdx), %xmm0           ## xmm0 = mem[0],zero
    vmovsd  8(%rdx), %xmm7          ## xmm7 = mem[0],zero
    vmovsd  16(%rdx), %xmm6         ## xmm6 = mem[0],zero
    vmovsd  16(%rsi), %xmm11        ## xmm11 = mem[0],zero
    vmovsd  40(%rsi), %xmm12        ## xmm12 = mem[0],zero
    vmovsd  64(%rsi), %xmm9         ## xmm9 = mem[0],zero
    vmovsd  24(%rdx), %xmm3         ## xmm3 = mem[0],zero
    vmovupd (%rsi), %xmm4
    vmovupd 8(%rsi), %xmm10
    vmovhpd (%rsi), %xmm11, %xmm5   ## xmm5 = xmm11[0],mem[0]
    vinsertf128     $1, %xmm5, %ymm4, %ymm5
    vunpcklpd       %xmm3, %xmm0, %xmm1 ## xmm1 = xmm0[0],xmm3[0]
    vmovddup        %xmm0, %xmm0    ## xmm0 = xmm0[0,0]
    vinsertf128     $1, %xmm1, %ymm0, %ymm0
...
```

In the code above there is a lot of activity in the `xmm` registers (which are smaller than `ymm`).
indeed, if we benchmark the 3x3 matrix multiply we find that it is in fact slower than the 4x4 version (note that in the
benchmark below the matrix is wrapped in a `Ref` here to prevent the compiler from constant folding the benchmark loop):

```jl
julia> using BenchmarkTools # import Pkg; Pkg.add("BenchmarkTools") to install

julia> for n in (2,3,4)
           s = Ref(rand(SMatrix{n,n}))
           @btime $(s)[] * $(s)[]
       end
  2.711 ns (0 allocations: 0 bytes)
  10.273 ns (0 allocations: 0 bytes)
  6.059 ns (0 allocations: 0 bytes)
```

In these cases, we can explicitly vectorize the code using the SIMD vector library [SIMD.jl](https://github.com/eschnett/SIMD.jl).
SIMD.jl provides a type `Vec{N, T}` where `N` is the number of elements and `T` is the element type.
`Vec{N, T}` is similar to the LLVM `<N x T>` vector type and operations on `Vec` typically translate directly to LLVM operations:

For example, below we define some input data and a function `g` that do some simple arithmetic. We then look at the generated code.

```jl
julia> using SIMD

julia> a = Vec((1,2,3,4))
<4 x Int64>[1, 2, 3, 4]

julia> b = Vec((1,2,3,4))
<4 x Int64>[1, 2, 3, 4]

julia> g(a, b, c) = a * b + c;

julia> @code_llvm g(a, b, 3)
...
  %res.i = mul <4 x i64> %7, %6
  %8 = insertelement <4 x i64> undef, i64 %3, i32 0
  %9 = shufflevector <4 x i64> %8, <4 x i64> undef, <4 x i32> zeroinitializer
  %res.i1 = add <4 x i64> %res.i, %9
...
```

The `mul <4 x i64>` is the multiplication of the two vectors, and then the scalar `3` is "broadcasted" to a vector
and added to the result. Feel free to look at `@code_native` to see the native SIMD instructions.
We cand use SIMD.jl to write a faster 3x3 matrix multiplication:

```
function matmul3x3(a::SMatrix, b::SMatrix)
    D1 = a.data; D2 = b.data
    # Extract data from matrix into SIMD.jl Vec
    SV11 = Vec((D1[1], D1[2], D1[3]))
    SV12 = Vec((D1[4], D1[5], D1[6]))
    SV13 = Vec((D1[7], D1[8], D1[9]))

    # Form the columns of the resulting matrix
    r1 = muladd(SV13, D2[3], muladd(SV12, D2[2], SV11 * D2[1]))
    r2 = muladd(SV13, D2[6], muladd(SV12, D2[5], SV11 * D2[4]))
    r3 = muladd(SV13, D2[9], muladd(SV12, D2[8], SV11 * D2[7]))

    return SMatrix{3,3}((r1[1], r1[2], r1[3],
                         r2[1], r2[2], r2[3],
                         r3[1], r3[2], r3[3]))
end
```

Let's compare this new version to the default the 3x3 version:

```jl
julia> s = Ref(rand(SMatrix{n,n}))

julia> @btime $(s)[] * $(s)[];
  10.281 ns (0 allocations: 0 bytes)

julia> @btime matmul3x3($(s)[], $(s)[]);
  4.392 ns (0 allocations: 0 bytes)
```

A guite significant improvement! The code for `matmul3x3` could, of course, be generalized to work for more sizes, perhaps using a `@generated` function.

## Using intrinsics

All we have done so far has been architecture independent.
If the CPU only supports an old version of SIMD or perhaps doesn't support SIMD at all, LLVM will just compile the code
using the latest features that are available, falling back to scalar instructions if needed.
However, in some cases, we really do want to use a specific instruction in a certain instruction set supported by the CPU.
The idea for writing this blog post was from reading about
a new hashing library called ["meowhash"](https://github.com/cmuratori/meow_hash) was released.
It uses AES decryption which processors now has built-in instructions to perform.
Looking in the [source code](https://github.com/cmuratori/meow_hash/blob/7a871d7edf4405c2ee361d1401a1eb395926fcca/meow_intrinsics.h#L93)
we can see the macro:

```cpp
#define Meow128_AESDEC(Prior, Xor) _mm_aesdec_si128((Prior), (Xor))
```

In the [Intel Intrinsics Guide](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#text=_mm_aesdec_si128&expand=221)
this intrinsic is described as

> Perform one round of an AES decryption flow on data (state) in a using the round key in `RoundKey`, and store the result in `dst`.

If we wanted to port meow-hash to Julia we would need to call this intrinsic in Julia, so how should we do that?

Firstly, Julia allows calling LLVM intrinsics through `ccall`.
We can for example call the `pow` intrinsic for two `Float64` as:

```jl
julia> llvm_pow(a, b) = ccall("llvm.pow.f64", llvmcall, Float64, (Float64, Float64), a, b);

julia> llvm_pow(2.0, 3.0)
8.0
```

So, in order to call our AES decryption instruction, we need to know the corresponding LLVM intrinsic to `_mm_aesdec_si128`. Since Julia itself doesn't provide us
with a way to get the intrinsic,
we need to ask a compiler that does. Fortunately, we can just ask Clang to emit the corresponding LLVM for us.
Using the [Godbolt compiler webtool](https://godbolt.org/z/vBTDVy) makes this very easy.
In the link to Godbolt we can see the following (slightly cleaned up):

```
define <2 x i64> @_Z6aesdecDv2_xS_(<2 x i64>, <2 x i64>) local_unnamed_addr #0 !dbg !262 {
  %3 = call <2 x i64> @llvm.x86.aesni.aesdec(<2 x i64> %0, <2 x i64> %1) #3, !dbg !270
  ret <2 x i64> %3, !dbg !271
}
```

So the intrinsic is called `x86.aesni.aesdec`. Now, we just need to know how to pass in the argument types which are
`<2 x i64>`. A normal Julia tuple of integers will not do because it gets passed to LLVM as an [`array`](https://llvm.org/docs/LangRef.html#array-type)
and not a `vector`
Instead, we need to send in a tuple with special elements of the type [`VecElement`](https://docs.julialang.org/en/v1/base/simd-types/).
Julia treats a tuple of `VecElement`s special and will pass it to LLVM as a `vector`.

All that is now left is to define some convenience typealias, create our inputs and call the intrinsic:

```jl
julia> const __m128i = NTuple{2, VecElement{Int64}};

julia> aesdec(a, roundkey) = ccall("llvm.x86.aesni.aesdec", llvmcall, __m128i, (__m128i, __m128i), a, roundkey);

julia> aesdec(__m128i((213132, 13131)), __m128i((31231, 43213)))
(VecElement{Int64}(-1627618977772868053), VecElement{Int64}(999044532936195731))
```

So now, we are in a position to port Meow Hash to Julia!

It should be stated that intrinsics should only be used as a last resort. It will lead to your code being less portable
and harder to maintain.

## Conclusion

There are many ways of doing SIMD in Julia. From letting the compiler to do the the job to using a SIMD library, and finally
getting our hands dirty and use the intrinsics. Which way is best will depend on your application but hopefully, this helped a bit
with showing what options are available.
