## behind the moar for performance ...

TypeProfiler.jl employs Julia's type inference for bug reports.

!!! note
    TypeProfiler.jl needs Julia v1.6, especially [this commit](https://github.com/JuliaLang/julia/commit/d5cf73ffffbab40ae06cc1ec99cac9d8e3d2b6a2);
    as such I recommend you give a try on this package with [nighlty](https://julialang.org/downloads/nightlies/)
    or [building from source](https://github.com/JuliaLang/julia) after the commit.


### demo

Say you have this strange and buggy file and want to know where to fix:

> demo.jl

```julia
########
# demo #
########

# fibonacci
# ---------

fib(n) = n ≤ 2 ? n : fib(n-1) + fib(n-2)

fib(1000)   # never terminates in ordinal execution
fib(m)      # undef var
fib("1000") # obvious type error


# language features
# -----------------

# user-defined types
struct Ty{T}
    fld::T
end

function foo(a)
    v = Ty(a)
    return bar(v)
end

# macros will be expanded
@inline bar(n::T)     where {T<:Number} = n < 0 ? zero(T) : one(T)
@inline bar(v::Ty{T}) where {T<:Number} = bar(v.fdl) # typo "fdl"
@inline bar(v::Ty)                      = bar(convert(Number, v.fld))

foo(1.2)
foo("1") # `String` can't be converted to `Number`
```

You can have TypeProfiler.jl detect possible errors:

```julia
julia> using TypeProfiler

julia> profile_and_watch_file("demo.jl")

profiling from demo.jl ... (finished in 2.745 sec)
═════ 6 possible errors found ═════
┌ @ demo.jl:11 Main.fib(Main.m)
│ variable Main.m is not defined: Main.fib(Main.m)
└
┌ @ demo.jl:12 Main.fib("1000")
│┌ @ demo.jl:8 Main.≤(n::String, 2)
││┌ @ operators.jl:326 Base.<(x::String, y::Int64)
│││┌ @ operators.jl:277 Base.isless(x::String, y::Int64)
││││ no matching method found for call signature: Base.isless(x::String, y::Int64)
│││└
│┌ @ demo.jl:8 Main.-(n::String, 1)
││ no matching method found for call signature: Main.-(n::String, 1)
│└
│┌ @ demo.jl:8 Main.-(n::String, 2)
││ no matching method found for call signature: Main.-(n::String, 2)
│└
┌ @ demo.jl:33 Main.foo(1.2)
│┌ @ demo.jl:25 Main.bar(v::Union{})
││┌ @ demo.jl:30 Base.getproperty(v::Main.Ty{Float64}, :fdl::Symbol)
│││┌ @ Base.jl:33 Base.getfield(x::Main.Ty{Float64}, f::Symbol)
││││ invalid builtin function call: Base.getfield(x::Main.Ty{Float64}, f::Symbol)
│││└
┌ @ demo.jl:34 Main.foo("1")
││┌ @ demo.jl:31 Main.convert(Main.Number, %2::String)
│││ no matching method found for call signature: Main.convert(Main.Number, %2::String)
││└
```

Hooray !
TypeProfiler.jl found possible error points (e.g. `MethodError: no method matching isless(::String, ::Int64)`) given toplevel call signatures of generic functions (e.g. `fib("1000")`).

Note that TP can find these errors while demo.jl is so ridiculous (especially the `fib` implementation) that it would never terminate in actual execution.
That is possible because TP profiles code only on _type level_.
This technique is often called "abstract interpretation" and TP internally uses Julia's native abstract interpreter implementation (for its JIT compile), so it can profile code as fast/correctly as Julia's code generation.

Lastly let's apply the following diff to demo.jl so that it works nicely:

> fix-demo.jl.diff

```diff
diff --git a/demo.jl b/demo-fixed.jl
index d2b188a..1d1b3da 100644
--- a/demo.jl
+++ b/demo.jl
@@ -5,11 +5,21 @@
 # fibonacci
 # ---------

-fib(n) = n ≤ 2 ? n : fib(n-1) + fib(n-2)
+# cache, cache, cache
+function fib(n::T) where {T<:Number}
+    cache = Dict(zero(T)=>zero(T), one(T)=>one(T))
+    return _fib(n, cache)
+end
+_fib(n, cache) = if haskey(cache, n)
+    cache[n]
+else
+    cache[n] = _fib(n-1, cache) + _fib(n-2, cache)
+end

-fib(1000)   # never terminates in ordinal execution
-fib(m)      # undef var
-fib("1000") # obvious type error
+fib(BigInt(1000)) # will terminate in ordinal execution as well
+m = 1000          # define m
+fib(m)
+fib(parse(Int, "1000"))


 # language features
@@ -27,8 +37,8 @@ end

 # macros will be expanded
 @inline bar(n::T)     where {T<:Number} = n < 0 ? zero(T) : one(T)
-@inline bar(v::Ty{T}) where {T<:Number} = bar(v.fdl) # typo "fdl"
+@inline bar(v::Ty{T}) where {T<:Number} = bar(v.fld) # typo fixed
 @inline bar(v::Ty)                      = bar(convert(Number, v.fld))

 foo(1.2)
-foo("1") # `String` can't be converted to `Number`
+foo('1') # `Char` will be converted to `UInt32`
```

If you apply the diff (i.e. update and save the demo.jl), TP will automatically re-trigger profiling, and this time, won't complain anything:

> git apply fix-demo.diff

```julia
profiling from demo.jl ... (finished in 5.146 sec)
No errors !
```


### TODOs

- support `module` expressions, profiling on a package (without actual loading)
- more reports
  * more correct error reports in general
  * report some cases of `throw`, e.g. `rand('1')::ArgumentError("Sampler for this object is not defined")`
- balance between Julia's inference approach and error profiling ?
  - Julia's type inference allows abstract type (like `Any`) to slip into the inference process by various heuristics, in order to ensure its termination and obtain the performance
  - but this is somewhat unideal in the context of bug reports, since the stance would be _"better safe than sorry"_, meaning we ideally want to find all the possible errors while revealing some uncertainty Julia's inference accepts
- report performance pitfalls ?
