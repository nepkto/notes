# Runtime-compiled Regex vs Source-generated Regex (`GeneratedRegex`) — Detailed Note

This note explains the practical differences between a **runtime-compiled regex** (created using `new Regex(...)` during execution) and a **source-generated regex** (created at build time using the `[GeneratedRegex]` attribute).

---

## 1) Runtime-compiled regex (created at execution time)

### What it is
A runtime-compiled regex is constructed when the code runs:

```csharp
var regex = new Regex(@"\t|\n|\r", RegexOptions.None, TimeSpan.FromSeconds(5));
```

### What happens at runtime
When `new Regex(...)` executes, .NET typically performs:

- **Parsing** the pattern (`\t|\n|\r`) into an internal representation.
- **Building an execution plan** (an internal state machine).
- **Allocations** for the `Regex` object and its internal data structures.
- Potential **cache interaction** (there is a runtime cache for some regex usage patterns, but it doesn’t eliminate costs when you instantiate new `Regex` repeatedly).

### Why it can be slower in hot paths
If this code sits in a frequently executed method, repeatedly calling `new Regex(...)` can cause:

- Repeated **CPU work** (parsing/initialization).
- Repeated **GC pressure** (object allocations).
- More time spent inside regex setup rather than doing the actual string replacement.

### Mitigation (without source generation)
Even without `[GeneratedRegex]`, a common improvement is to cache the regex instance:

```csharp
private static readonly Regex LineEndingsRegex =
    new(@"\t|\n|\r", RegexOptions.None, TimeSpan.FromSeconds(5));
```

This avoids repeated instantiation but still relies on runtime parsing/building at process start.

---

## 2) Source-generated regex (`[GeneratedRegex]`) (created at build time)

### What it is
A source-generated regex uses .NET’s regex source generator. You declare the pattern via an attribute:

```csharp
[GeneratedRegex(@"\t|\n|\r", RegexOptions.None, 5000)]
private static partial Regex LineEndingsRegex();
```

The compiler/source generator produces C# code for the regex implementation during the build, and `LineEndingsRegex()` returns a cached instance.

### What happens at build time
- The regex is **analyzed and code-generated** during compilation.
- The emitted code includes an optimized implementation of the pattern.
- The runtime no longer needs to parse the pattern text on the hot path.

### Benefits in runtime behavior
- **Lower startup and per-call overhead**: minimal/no runtime parsing for the regex pattern.
- **Fewer allocations** and less GC pressure compared to repeatedly constructing `Regex`.
- Often **faster execution** because the generated implementation can avoid some of the general-purpose regex machinery.

### Tradeoffs / considerations
- Requires a .NET version that supports `GeneratedRegex` (modern .NET SDKs).
- The pattern becomes part of generated code, which can increase IL/code size slightly (usually acceptable and worth it for hot paths).
- Some very dynamic scenarios (patterns built at runtime) can’t use source generation.

---

## 3) Key differences at a glance

- **When pattern is processed**
  - Runtime regex: pattern processed at runtime (parsing/building).
  - GeneratedRegex: pattern processed at compile time (source generation).

- **Allocation pattern**
  - Runtime regex created in method: allocations each call.
  - GeneratedRegex: cached instance returned; avoids repeated allocations.

- **Best use cases**
  - Runtime regex: dynamic patterns, one-off use, low-frequency code paths.
  - GeneratedRegex: static patterns used frequently, performance-sensitive paths.

---

## 4) In our change: why it matters

We replaced:

- Creating `new Regex(...)` inside `ProcessStringValue` (per invocation)

with:

- Calling a generated, cached regex (`LineEndingsRegex()`)

This specifically targets:
- Reduced repeated initialization work
- Reduced allocations/GC pressure
- Better throughput for repeated string sanitization (replacing `\t`, `\n`, `\r` with spaces)

---