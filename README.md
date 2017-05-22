# A lens into the soul of your program

[![Build Status](https://travis-ci.org/zenna/Lens.jl.svg?branch=master)](https://travis-ci.org/zenna/Lens.jl)

Lens.jl is a simple Julia package which makes it easy to dynamically inspect and extract values deep within a program, with minimal interference to the program itself.

The philosophy of Lens is that observation should not imply interference.  A running program is like a machine; there are many possible things we might like to know about its behaviour, but we want a clean interface that doesn't require us to mutate our machine in order to observe it.

# Installation

Lens is in the official Julia Package repository.  You can easily install it from a Julia REPL with:

```julia
Pkg.add("Lens")
using Lens
```

# Usage

Suppose we have a function which [bubble sorts](http://en.wikipedia.org/wiki/Bubble_sort) an array:

```julia
function bubblesort{T}(a::AbstractArray{T,1})
    b = copy(a)
    isordered = false
    span = length(b)
    i = 0
    while !isordered && span > 1
        lens(:start_of_loop, b, i) # <--- lens here!!
        isordered = true
        for i in 2:span
            if b[i] < b[i-1]
                t = b[i]
                b[i] = b[i-1]
                b[i-1] = t
                isordered = false
            end
        end
        span -= 1
        i += 1
    end
    lens(:after_loop, sorteddata=b, niters=i) # <--- and here!!
    return b
end
```

The algorithm details don't matter; what's important is the `lens`.  Lenses are created in one of two forms.  The first form, as used on line 7, is as follows:

```julia
lens(lensname::Symbol, x, y, ...)
```

The first argument is a Symbol and gives a name to the lens.  We'll need to remember the name for later when we attach *listeners* to the lens.
The remaining arguments `x,y,...` are any values you want the lens to capture.

Lenses capture values we specify, then propagate that data onto `Filters`.
Lenses themselves do not contain any information about the listeners, the listeners are attached onto Lens through a function `register!`.  For example we can register a printing listener to the lens with:

```julia
register!(println, :print_data, :start_of_loop, false)
```

Then if we simply call `bubblesort` our lens will be activated.

```julia
julia> bubblesort([1,2,0,3])
[1,2,0,3]
[1,0,2,3]
[0,1,2,3]
4-element Array{Int64,1}:
 0
 1
 2
 3
```

The previous call to register used the method `register!(f::Function, listenername::Symbol, lensname::Symbol, kwargs::Bool)`.  It took a  function as input and automatically created a `Filter` for us.  We can of course make the `Filter` explicitly:

```julia
register!(:start_of_loop, Filter(:print_data, print, true, true))
```

The arguments are:

1. the listeners name (which might be useful if we want to remove or disable the listener later)
2. a function which transforms information sent to it by the lens
3. whether the listener is enabled or not
4. whether the listener takes keyworld arguments (we'll get to this soon)

But the function form is often more convenient, and it allows us us to use the `do` notation. E.g.:

```julia
julia> clear_all_listeners!()
julia> register!(:print_data, :start_of_loop, false) do data, index
         println("on $index-th iteration the first element is $(index[1])")
       end
Set{Filter}({Filter(:print_data,(anonymous function),true,false)})

julia> bubblesort([50,2,0,20])
on 0-th iteration the first element is 0
on 1-th iteration the first element is 1
on 2-th iteration the first element is 2
4-element Array{Int64,1}:
  0
  2
 20
 50
```

## Keyword arguments

The second lens in our example uses keyword arguments.  Keyword arguments are useful because often as we develop our programs, we will find we want to capture to new pieces of data.  Rather than modifying all our Filters, we instead just add on the data to with a new keyword.

For example, suppose we decided now that we wanted to capture the input Array as well as the sorted array, we would simply change the lens to

`lens(:after_loop, inputdata=a, sorteddata=b, niters=i)`

__Note__: A function used in a listener that takes input from a keyword argument lens will recieve a single input.  It can extract the relevant fields using `data[:key]` syntax.

## Capturing

Often we want to just extract or capture some values somewhere deep within our programs.  Suppose we're interested in the distributon over the number of loop iterations.  Let's extend our example so to randomly generate 1000 vectors and bubblesorts them all.

```julia
function many_bubbles()
  for i = 1:1000
    bubblesort(rand(1000))
  end
end
```

`capture` retruns a pair `(value, result)` where `value` is the value returned from calling the function passed into capture, and `result` is a  datastructure which contains the captured information.  A `result` is a little complicated as a datastructure, but we can easily extract the data we need using `get`.

```julia
julia> get(iters)
1000-element Array{Any,1}:
 88
 88
 95
 77
 94
 83
 84
 89
 84
 81
 90
 83
 82
 95
 97
 89
  ⋮
 80
 93
 94
 85
 79
 96
 91
 86
 89
 93
 98
 96
 87
 88
 96
 95
```

<!-- ```julia
julia> using Gadfly
julia> plot(x=get(iters),Geom.density)
```
![iteration_distribution](images/density.svg?raw=true) -->

## Enabling and Disabling Filters

If you want to temporarily disable all the listeners and render all your lenses ineffective use `disable_all_listeners!()`.  To re-enable them all use `enable_all_listeners!()`.  These can be convenient for example if you want dynamically switch between listeners which are printing information to the screen.

For more fine grained control use enable `enable_listener!(watch_name::Symbol, listener_name::Symbol)` which enables the lens with name `watch_name` propagating to the listener with name `listener_name`.  Unsurprisingly, these is also `disable_listener!`.

For more permanent effect there is `delete_listener!` and `clear_all_listeners!`, which are just like above but not temporary.