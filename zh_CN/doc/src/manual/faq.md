# 常见问题

## 会话和 REPL

### 如何从内存中删除某个对象？

Julia 没有类似于 MATLAB 的 `clear` 函数，某个名称一旦定义在 Julia 的会话中（准确地说，在 `Main` 模块中），它就会一直存在下去。

如果关心内存用量，一个对象总能被一个占用更少内存的对象替换掉。例如，如果 `A` 是一个不再需要的GB量级的数组，可以使用 `A = nothing` 来释放内存。该内存将在下一次垃圾回收器运行时被释放，也可以使用 [`gc()`](@ref Base.GC.gc) 强制执行。另外，试图使用 `A` 很可能导致错误，因为大部分方法（method）在 `Nothing` 类型上没有定义。

### 如何在会话中修改某个类型的声明？

也许你定义了某个类型，后来发现需要向其中增加一个新的域。如果在 REPL 中尝试这样做，会得到一个错误：

```
ERROR: invalid redefinition of constant MyType
```

模块 `Main` 中的类型不能重新定义。

尽管这在开发新代码时会造成不便，但是对于这个问题仍然有一个不错的解决办法：可以用重新定义的模块替换原有的模块，所以可以把所有新代码封装在一个模块里，然后重新定义类型和常量。虽说不能将类型名称导入到 `Main` 模块中再去重新定义，但是可以用模块名来改变作用范围。换言之，开发时的工作流可能类似这样：

```julia
include("mynewcode.jl") # this defines a module MyModule
obj1 = MyModule.ObjConstructor(a, b)
obj2 = MyModule.somefunction(obj1)
# Got an error. Change something in "mynewcode.jl"
include("mynewcode.jl") # reload the module
obj1 = MyModule.ObjConstructor(a, b) # old objects are no longer valid, must reconstruct
obj2 = MyModule.somefunction(obj1) # this time it worked!
obj3 = MyModule.someotherfunction(obj2, c)
...
```

## 函数

### 向函数传递了参数 `x`，在函数中做了修改，但是在函数外变量 `x` 的值还是没有变。为什么？

假设函数被如此调用：

```jldoctest
julia> x = 10
10

julia> function change_value!(y)
           y = 17
       end
change_value! (generic function with 1 method)

julia> change_value!(x)
17

julia> x # x is unchanged!
10
```

在 Julia 中，通过将 `x` 作为参数传递给函数，不能改变变量`x`的绑定。在上例中，调用 `change_value!(x)` 时，`y` 是一个新建变量，初始时与 `x` 的值绑定，即 `10`。然后 `y` 与常量 `17` 重新绑定，此时变量外作用域中的 `x` 并没有变动。

但是这里有一个需要注意的点：假设 `x` 被绑定至 `Array` 类型 (或者其他 *可变* 的类型)。在函数中，你无法将 `x` 与 Array “解绑”，但是你可以改变其内容。例如：

```jldoctest
julia> x = [1,2,3]
3-element Array{Int64,1}:
 1
 2
 3

julia> function change_array!(A)
           A[1] = 5
       end
change_array! (generic function with 1 method)

julia> change_array!(x)
5

julia> x
3-element Array{Int64,1}:
 5
 2
 3
```

这里我们新建了一个函数 `chang_array!`，它把 `5` 赋值给传入的数组（在调用处与 `x` 绑定，在函数中与 `A` 绑定）的第一个元素。注意，在函数调用之后，`x` 依旧与同一个数组绑定，但是数组的内容变化了：变量 `A` 和 `x` 是不同的绑定，引用同一个可变的 `Array` 对象。

### 函数内部能否使用 `using` 或 `import`？

不可以，不能在函数内部使用 `using` 或 `import` 语句。如果你希望导入一个模块，但只在特定的一个或一组函数中使用它的符号，有以下两种方式：

1. 使用 `import`：

   ```julia
   import Foo
   function bar(...)
       # ... refer to Foo symbols via Foo.baz ...
   end
   ```

   这会加载 `Foo` 模块，同时定义一个变量 `Foo` 引用该模块，但并不会
   将其他任何符号从该模块中导入当前的命名空间。
   `Foo` 等符号可以由限定的名称 `Foo.bar` 等引用等。
2. 将函数封装到模块中：

   ```julia
   module Bar
   export bar
   using Foo
   function bar(...)
       # ... refer to Foo.baz as simply baz ....
   end
   end
   using Bar
   ```

   这会从 `Foo` 中导入所有符号，但仅限于 `Bar` 模块内。

### 运算符 `...` 有何作用？

### `...`运算符的两个用法：slurping和splatting

很多Julia的新手会对运算符`...`的用法感到困惑。让`...`用法如此困惑的部分原因是根据上下文它有两种不同的含义。

### `...`在函数定义中将多个参数组合成一个参数

在函数定义的上下文中，`...`运算符用来将多个不同的参数组合成单个参数。`...`运算符的这种将多个不同参数组合成单个参数的用法称为slurping：

```jldoctest
julia> function printargs(args...)
           println(typeof(args))
           for (i, arg) in enumerate(args)
               println("Arg #$i = $arg")
           end
       end
printargs (generic function with 1 method)

julia> printargs(1, 2, 3)
Tuple{Int64,Int64,Int64}
Arg #1 = 1
Arg #2 = 2
Arg #3 = 3
```

如果Julia是一个使用ASCII字符更加自由的语言的话，slurping运算符可能会写作`<-...`而非`...`。

### `...`在函数调用中将一个参数分解成多个不同参数

与在定义函数时表示将多个不同参数组合成一个参数的`...`运算符用法相对，当用在函数调用的上下文中`...`运算符也用来将单个的函数参数分成多个不同的参数。`...`函数的这个用法叫做splatting：

```jldoctest
julia> function threeargs(a, b, c)
           println("a = $a::$(typeof(a))")
           println("b = $b::$(typeof(b))")
           println("c = $c::$(typeof(c))")
       end
threeargs (generic function with 1 method)

julia> x = [1, 2, 3]
3-element Array{Int64,1}:
 1
 2
 3

julia> threeargs(x...)
a = 1::Int64
b = 2::Int64
c = 3::Int64
```

如果Julia是一个使用ASCII字符更加自由的语言的话，splatting运算符可能会写作`...->`而非`...`。

### 赋值语句的返回值是什么？

`=`运算符始终返回右侧的值，所以：

```jldoctest
julia> function threeint()
           x::Int = 3.0
           x # returns variable x
       end
threeint (generic function with 1 method)

julia> function threefloat()
           x::Int = 3.0 # returns 3.0
       end
threefloat (generic function with 1 method)

julia> threeint()
3

julia> threefloat()
3.0
```

相似地：

```jldoctest
julia> function threetup()
           x, y = [3, 3]
           x, y # returns a tuple
       end
threetup (generic function with 1 method)

julia> function threearr()
           x, y = [3, 3] # returns an array
       end
threearr (generic function with 1 method)

julia> threetup()
(3, 3)

julia> threearr()
2-element Array{Int64,1}:
 3
 3
```

## 类型，类型声明和构造函数

### [何谓“类型稳定”？](@id man-type-stability)

这意味着输出的类型可以由输入的类型预测出来。特别地，这意味着输出的类型不会因输入的*值*的不同而变化。以下代码*不是*类型稳定的：

```jldoctest
julia> function unstable(flag::Bool)
           if flag
               return 1
           else
               return 1.0
           end
       end
unstable (generic function with 1 method)
```

根据参数的不同，该函数可能返回 `Int` 或 [`Float64`](@ref)。
由于 Julia 无法在编译期预测该函数的返回值类型，任何使用该函数的计算都需要考虑这两种可能的返回类型，这样难以生成高效的机器码。

### [为何 Julia 对某个看似合理的操作返回 `DomainError`？](@id faq-domain-errors)

某些运算在数学上有意义，但会产生错误：

```jldoctest
julia> sqrt(-2.0)
ERROR: DomainError with -2.0:
sqrt will only return a complex result if called with a complex argument. Try sqrt(Complex(x)).
Stacktrace:
[...]
```

这一行为是为了保证类型稳定而带来的不便。对于 [`sqrt`](@ref)，许多用户会希望 `sqrt(2.0)` 产生一个实数，如果得到了复数 `1.4142135623730951 + 0.0im` 则会不高兴。也可以编写 [`sqrt`](@ref) 函数，只有当传递一个负数时才切换到复值输出，但结果将不是[类型稳定](@ref man-type-stability)的，而且 [`sqrt`](@ref) 函数的性能会很差。

在这样那样的情况下，若你想得到希望的结果，你可以选择一个*输入类型*，它可以使根据你的想法接受一个*输出类型*，从而结果可以这样表示：

```jldoctest
julia> sqrt(-2.0+0im)
0.0 + 1.4142135623730951im
```

### 为什么Julia使用原生机器整数算法？

Julia使用机器算法进行整数计算。这意味着`Int`的范围是有界的，值在范围的两端循环，也就是说整数的加法，减法和乘法会出现上溢或者下溢，导致出现某些从开始就令人不安的结果：

```jldoctest
julia> typemax(Int)
9223372036854775807

julia> ans+1
-9223372036854775808

julia> -ans
-9223372036854775808

julia> 2*ans
0
```

无疑，这与数学上的整数的行为很不一样，并且你会想对于高阶编程语言来说把这个暴露给用户难称完美。然而，对于效率优先和透明度优先的数值计算来说，其他的备选方案可谓更糟。

一个备选方案是去检查每个整数运算是否溢出，如果溢出则将结果提升到更大的整数类型比如[`Int128`](@ref)或者[`BigInt`](@ref)。 不幸的是，这会给所有的整数操作（比如让循环计数器自增）带来巨大的额外开销 — 这需要生成代码去在算法指令后进行运行溢出检测，并生成分支去处理潜在的溢出。更糟糕的是，这会让涉及整数的所有运算变得类型不稳定。如同上面提到的，对于高效生成高效的代码[类型稳定很重要](@ref man-type-stability)。如果不指望整数运算的结果是整数，就无法想C和Fortran编译器一样生成快速简单的代码。

这个方法有个变体可以避免类型不稳定的出现，这个变体是将类型`Int`和[`BigInt`](@ref)合并成单个混合整数类型，当结果不再满足机器整数的大小时会内部自动切换表示。虽然表面上在Julia代码层面解决了类型不稳定，但是这个只是通过将所有的困难硬塞给实现混合整数类型的C代码而掩盖了这个问题。这个方法*可能*有用，甚至在很多情况下速度很快，但是它有很多缺点。一个缺点是整数和整数数组的内存上的表示不再与C、Fortran和其他使用原生机器整数的怨言所使用的自然表示一样。所以，为了与那些语言协作，我们无论如何最终都需要引入原生整数类型。任何整数的无界表示都不会占用固定的比特数，所以无法使用固定大小的槽来内联地存储在数组中 — 大的整数值通常需要单独的堆分配的存储。并且无论使用的混合整数实现多么智能，总会存在性能陷阱 — 无法预期的性能下降的情况。复杂的表示，与C和Fortran协作能力的缺乏，无法在不使用另外的堆存储的情况下表示整数数组，和无法预测的性能特性让即使是最智能化的混合整数实现对于高性能数值计算来说也是个很差的选择。

除了使用混合整数和提升到BigInt，另一个备选方案是使用饱和整数算法，此时最大整数值加一个数时值保持不变，最小整数值减一个数时也是同样的。这就是Matlab™的做法：

```
>> int64(9223372036854775807)

ans =

  9223372036854775807

>> int64(9223372036854775807) + 1

ans =

  9223372036854775807

>> int64(-9223372036854775808)

ans =

 -9223372036854775808

>> int64(-9223372036854775808) - 1

ans =

 -9223372036854775808
```

乍一看，这个似乎足够合理，因为9223372036854775807比-9223372036854775808更接近于9223372036854775808并且整数还是以固定大小的自然方式表示的，这与C和Fortran相兼容。但是饱和整数算法是很有问题的。首先最明显的问题是这并不是机器整数算法的工作方式，所以实现饱和整数算法需要生成指令，在每个机器整数运算后检查上溢或者下溢并正确地讲这些结果用[`typemin(Int)`](@ref)或者[`typemax(Int)`](@ref)取代。单单这个就将整数运算从单语句的快速的指令扩展成六个指令，还可能包括分支。哎呦喂~~但是还有更糟的 — 饱和整数算法并不满足结合律。考虑下列的Matlab计算：

```
>> n = int64(2)^62
4611686018427387904

>> n + (n - 1)
9223372036854775807

>> (n + n) - 1
9223372036854775806
```

这就让写很多基础整数算法变得困难因为很多常用技术都是基于有溢出的机器加法*是*满足结合律这一事实的。考虑一下在Julia中求整数值`lo`和`hi`之间的中点值，使用表达式`(lo + hi) >>> 1`:

```jldoctest
julia> n = 2^62
4611686018427387904

julia> (n + 2n) >>> 1
6917529027641081856
```

看到了吗？没有任何问题。那就是2^62和2^63之间的正确地中点值，虽然`n + 2n`的值是 -4611686018427387904。现在使用Matlab试一下：

```
>> (n + 2*n)/2

ans =

  4611686018427387904
```

哎呦喂。在Matlab中添加`>>>`运算符没有任何作用，因为在将`n`与`2n`相加时已经破坏了能计算出正确地中点值的必要信息，已经出现饱和。

没有结合性不但对于不能依靠像这样的技术的程序员是不幸的，并且让几乎所有的希望优化整数算法的编译器铩羽而归。例如，因为Julia中的整数使用平常的机器整数算法，LLVM就可以自由地激进地优化像`f(k) = 5k-1`这样的简单地小函数。这个函数的机器码如下所示：

```julia-repl
julia> code_native(f, Tuple{Int})
  .text
Filename: none
  pushq %rbp
  movq  %rsp, %rbp
Source line: 1
  leaq  -1(%rdi,%rdi,4), %rax
  popq  %rbp
  retq
  nopl  (%rax,%rax)
```

这个函数的实际函数体只是一个简单地`leap`指令，可以立马计算整数乘法与加法。当`f`内联在其他函数中的时候这个更加有益：

```julia-repl
julia> function g(k, n)
           for i = 1:n
               k = f(k)
           end
           return k
       end
g (generic function with 1 methods)

julia> code_native(g, Tuple{Int,Int})
  .text
Filename: none
  pushq %rbp
  movq  %rsp, %rbp
Source line: 2
  testq %rsi, %rsi
  jle L26
  nopl  (%rax)
Source line: 3
L16:
  leaq  -1(%rdi,%rdi,4), %rdi
Source line: 2
  decq  %rsi
  jne L16
Source line: 5
L26:
  movq  %rdi, %rax
  popq  %rbp
  retq
  nop
```

因为`f`的调用内联化，循环体就只是简单地`leap`指令。接着，考虑一下如果循环迭代的次数固定的时候会发生什么：

```julia-repl
julia> function g(k)
           for i = 1:10
               k = f(k)
           end
           return k
       end
g (generic function with 2 methods)

julia> code_native(g,(Int,))
  .text
Filename: none
  pushq %rbp
  movq  %rsp, %rbp
Source line: 3
  imulq $9765625, %rdi, %rax    # imm = 0x9502F9
  addq  $-2441406, %rax         # imm = 0xFFDABF42
Source line: 5
  popq  %rbp
  retq
  nopw  %cs:(%rax,%rax)
```

因为编译器知道整数加法和乘法是满足结合律的并且乘法可以在加法上使用分配律 — 两者在饱和算法中都不成立 — 所以编译器就可以把整个循环优化到只有一个乘法和一个加法。饱和算法完全无法使用这种优化，因为在每个循环迭代中结合律和分配律都会失效导致不同的失效位置会得到不同的结果。编译器可以展开循环，但是不能代数上将多个操作简化到更少的等效操作。

让整数算法静默溢出的最合理的备用方案是所有地方都使用检查算法，当加法、减法和乘法溢出，产生不正确的值时引发错误。在[blog post](http://danluu.com/integer-overflow/)中，Dan Luu分析了这个方案，发现这个方案理论上的性能微不足道，但是最终仍然会消耗大量的性能因为编译器（LLVM和GCC）无法在加法溢出检测处优雅地进行优化。如果未来有所进步我们会考虑在Julia中默认设置为检查整数算法，但是现在，我们需要和溢出可能共同相处。

### 在远程执行中`UndefVarError`的可能原因有哪些？

如同这个错误表述的，远程结点上的`UndefVarError`的直接原因是变量名的绑定并不存在。让我们探索一下一些可能的原因。

```julia-repl
julia> module Foo
           foo() = remotecall_fetch(x->x, 2, "Hello")
       end

julia> Foo.foo()
ERROR: On worker 2:
UndefVarError: Foo not defined
Stacktrace:
[...]
```

闭包`x->x`中有`Foo`的引用，因为`Foo`在节点2上不存在，所以`UndefVarError`被扔出。

在模块中而非`Main`中的全局变量不会在远程节点上按值序列化。只传递了一个引用。新建全局绑定的函数（除了`Main`中）可能会导致之后扔出`UndefVarError`。

```julia-repl
julia> @everywhere module Foo
           function foo()
               global gvar = "Hello"
               remotecall_fetch(()->gvar, 2)
           end
       end

julia> Foo.foo()
ERROR: On worker 2:
UndefVarError: gvar not defined
Stacktrace:
[...]
```

在上面的例子中，`@everywhere module Foo`在所有节点上定义了`Foo`。但是调用`Foo.foo()`在本地节点上新建了新的全局绑定`gvar`，但是节点2中并没有找到这个绑定，这会导致`UndefVarError`错误。

注意着并不适用于在模块`Main`下新建的全局变量。模块`Main`下的全局变量会被序列化并且在远程节点的`Main`下新建新的绑定。

```julia-repl
julia> gvar_self = "Node1"
"Node1"

julia> remotecall_fetch(()->gvar_self, 2)
"Node1"

julia> remotecall_fetch(varinfo, 2)
name          size summary
––––––––– –––––––– –––––––
Base               Module
Core               Module
Main               Module
gvar_self 13 bytes String
```

这并不适用于`函数`或者`结构体`声明。但是绑定到全局变量的匿名函数被序列化，如下例所示。

```julia-repl
julia> bar() = 1
bar (generic function with 1 method)

julia> remotecall_fetch(bar, 2)
ERROR: On worker 2:
UndefVarError: #bar not defined
[...]

julia> anon_bar  = ()->1
(::#21) (generic function with 1 method)

julia> remotecall_fetch(anon_bar, 2)
1
```

## 包和模块

### "using"和"import"的区别是什么？

只有一个区别，并且在表面上（语法层面）这个区别看来很小。`using`和`import`的区别是使用`using`时你需要写`function Foo.bar(..`来用一个新方法来扩展模块Foo的函数bar，但是使用`import Foo.bar`时，你只需要写`function bar(...`，会自动扩展模块Foo的函数bar。

这个区别足够重要以至于提供不同的语法的原因是你不希望意外地扩展一个你根本不知道其存在的函数，因为这很容易造成bug。对于使用像字符串后者整数这样的常用类型的方法最有可能出现这个问题，因为你和其他模块都可能定义了方法来处理这样的常用类型。如果你使用`import`，你会用你自己的新实现覆盖别的函数的`bar(s::AbstractString)`实现，这会导致做的事情天差地别（并且破坏模块Foo中其他的依赖于调用bar的函数的所有/大部分的将来的使用）。

## 空值与缺失值

### [在Julia中"null"，"空"或者"缺失"是怎么工作的?](@id faq-nothing)

不像其它很多语言（例如 C 和 Java），Julia 对象默认不能为"null"。当一个引用（变量，对象域，或者数组元素）没有被初始化，访问它会立即扔出一个错误。这种情况可以使用函数 [`isdefined`](@ref) 或者 [`isassigned`](@ref Base.isassigned) 检测到。

一些函数只为了其副作用使用，并不需要返回一个值。在这些情况下，约定的是返回 `nothing` 这个值，这只是 `Nothing` 类型的一个单例对象。这是一个没有域的一般类型；除了这个约定之外没有任何特殊点，REPL 不会为它打印任何东西。有些语言结构不会有值，也产生 `nothing`，例如 `if false; end`。

对于类型`T`的值`x`只会有时存在的情况，`Union{T,Nothing}`类型可以用作函数参数，对象域和数组元素的类型，与其他语言中的[`Nullable`, `Option` or `Maybe`](https://en.wikipedia.org/wiki/Nullable_type)相等。如果值本身可以是`nothing`(显然当`T`是`Any`时），`Union{Some{T}, Nothing}`类型更加准确因为`x == nothing`表示值的缺失，`x == Some(nothing)`表示与`nothing`相等的值的存在。[`something`](@ref)函数允许使用默认值的展开的`Some`对象，而非`nothing`参数。注意在使用`Union{T,Nothing}`参数或者域时编译器能够生成高效的代码。

在统计环境下表示缺失的数据（R 中的 `NA` 或者 SQL 中的 `NULL`）请使用 [`missing`](@ref) 对象。请参照[`缺失值`](@ref missing)章节来获取详细信息。

空元组（`()`）是空值的另一个表示方式。但是这不应该真的被认为是空值，而应被认为是零值的元组。

空（或者"底层"）类型，写作`Union{}`（空的union类型）是没有值和子类型（除了自己）的类型。通常你没有必要用这个类型。


### 该如何检查当前文件是否正在以主脚本运行？

当一个文件通过使用`julia file.jl`来当做主脚本运行时，有人也希望激活另外的功能例如命令行参数操作。确定文件是以这个方式运行的一个方法是检查`abspath(PROGRAM_FILE) == @__FILE__`是不是`true`。

## 内存

### 为什么当`x`和`y`都是数组时`x += y`还会申请内存？

在 Julia 中，`x += y` 在语法分析中会用 `x = x + y` 代替。对于数组，结果就是它会申请一个新数组来存储结果，而非把结果存在 `x` 同一位置的内存上。

这个行为可能会让一些人吃惊，但是这个结果是经过深思熟虑的。主要原因是Julia中的不可变对象，这些对象一旦新建就不能改变他们的值。实际上，数字是不可变对象，语句`x = 5; x += 1`不会改变`5`的意义，改变的是与`x`绑定的值。对于不可变对象，改变其值的唯一方法是重新赋值。

为了稍微详细一点，考虑下列的函数：

```julia
function power_by_squaring(x, n::Int)
    ispow2(n) || error("This implementation only works for powers of 2")
    while n >= 2
        x *= x
        n >>= 1
    end
    x
end
```

在`x = 5; y = power_by_squaring(x, 4)`调用后，你可以得到期望的结果`x == 5 && y == 625`。然而，现在假设当`*=`与矩阵一起使用时会改变左边的值，这会有两个问题：

  * 对于普通的方阵，`A = A*B` 不能在没有临时存储的情况下实现：`A[1,1]`
    会被计算并且在被右边使用完之前存储在左边。
  * 假设你愿意申请一个计算的临时存储（这会消除
    `*=`就地计算的大部分要点）；如果你利用了`x`的可变性，
    这个函数会对于可变和不可变的输入有不同的行为。特别地，
    对于不可变的`x`，在调用后（通常）你会得到`y != x`，而对可变的`x`，你会有`y == x`。

因为支持范用计算被认为比能使用其他方法完成的潜在的性能优化（比如使用显式循环）更加重要，所以像`+=`和`*=`运算符以绑定新值的方式工作。

## 异步IO与并发同步写入

### 为什么对于同一个流的并发写入会导致相互混合的输出？

虽然流式 I/O 的 API 是同步的，底层的实现是完全异步的。

思考一下下面的输出：

```jldoctest
julia> @sync for i in 1:3
           @async write(stdout, string(i), " Foo ", " Bar ")
       end
123 Foo  Foo  Foo  Bar  Bar  Bar
```

这是因为，虽然`write`调用是同步的，每个参数的写入在等待那一部分I/O完成时会生成其他的Tasks。

`print`和`println`在调用中会"锁定"该流。因此把上例中的`write`改成`println`会导致：

```jldoctest
julia> @sync for i in 1:3
           @async println(stdout, string(i), " Foo ", " Bar ")
       end
1 Foo  Bar
2 Foo  Bar
3 Foo  Bar
```

你可以使用`ReentrantLock`来锁定你的写入，就像这样：

```jldoctest
julia> l = ReentrantLock()
ReentrantLock(nothing, Condition(Any[]), 0)

julia> @sync for i in 1:3
           @async begin
               lock(l)
               try
                   write(stdout, string(i), " Foo ", " Bar ")
               finally
                   unlock(l)
               end
           end
       end
1 Foo  Bar 2 Foo  Bar 3 Foo  Bar
```

## 数组

### 零维数组和标量之间的有什么差别？

零维数组是`Array{T,0}`形式的数组，它与标量的行为相似，但是有很多重要的不同。这值得一提，因为这是使用数组的范用定义来解释也符合逻辑的特殊情况，虽然最开始看起来有些非直觉。下面一行定义了一个零维数组：

```
julia> A = zeros()
0-dimensional Array{Float64,0}:
0.0
```

在这个例子中，`A`是一个含有一个元素的可变容器，这个元素可以通过`A[] = 1.0`来设置，通过`A[]`来读取。所有的零维数组都有同样的大小（`size(A) == ()`）和长度（`length(A) == 1`）。特别地，零维数组不是空数组。如果你觉得这个非直觉，这里有些想法可以帮助理解Julia的这个定义。

* 类比的话，零维数组是"点"，向量是"线"而矩阵
  是"面"。就像线没有面积一样（但是也能代表事物的一个集合）,
  点没有长度和任意一个维度（但是也能表示一个事物）。
* 我们定义`prod(())`为1，一个数组中的所有的元素个数是
  大小的乘积。零维数组的大小为`()`，所以
  它的长度为`1`。
* 零维数组原生没有任何你可以索引的维度
  -- 它们仅仅是`A[]`。我们可以给它们应用同样的"trailing one"规则，
  如同所有其他的数组维度一样，所以你实际上可以使用
  `A[1]`，`A[1,1]`等来索引

理解它与普通的标量之间的区别也很重要。标量不是一个可变的容器（尽管它们是可迭代的，可以定义像`length`，`getindex`这样的东西，*例如*`1[] == 1`）。特别地，如果`x = 0.0`是以一个标量来定义，尝试通过`x[] = 1.0`来改变它的值会报错。标量`x`能够通过`fill(x)`转化成包含它的零维数组，并且相对地，一个零维数组`a`可以通过`a[]`转化成其包含的标量。另外一个区别是标量可以参与到线性代数运算中，比如`2 * rand(2,2)`，但是零维数组的相似操作`fill(2) * rand(2,2)`会报错。

## Julia 版本发布

### 应该使用 Julia 的正式版（release version），测试版（beta version）还是每夜更新版（nightly version）？

如果您正在寻找稳定的代码库，您可能更喜欢Julia的正式版。 通常每6个月发布一次，为您提供编写代码的稳定平台。

如果您不介意稍微落后于最新的错误修正和更改，觉得稍微更快的修改更具吸引力，您可能更喜欢Julia的测试版。 此外，这些二进制文件在发布之前会进行测试，以确保它们完全正常运行。

如果您想利用该语言的最新更新，并且不介意今天可用的版本偶尔出现实际上并没有正常工作，您可能更喜欢 Julia 的每夜更新版。

最后，您也可以考虑自己从源代码编译Julia。 此选项主要适用于那些适应命令行或对学习感兴趣的人。 如果您是这样，您可能也有兴趣阅读我们的[贡献指南](https://github.com/JuliaLang/julia/blob/master/CONTRIBUTING.md)。

可以在[https://julialang.org/downloads/](https://julialang.org/downloads/)的下载页面上找到每种下载类型的链接。 请注意，并非所有版本的Julia都适用于所有平台。

### 已弃用的功能会在何时移除？

已弃用的函数会被随后发布的版本中移除。例如，在1.0版本中被标记为已弃用的函数会在0.2版本及之后的版本中无法使用。
