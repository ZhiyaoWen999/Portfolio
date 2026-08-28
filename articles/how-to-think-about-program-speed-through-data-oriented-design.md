# How to Think About Program Speed Through Data-Oriented Design

The main questions I am trying to understand are not just about writing faster code. They are about building a better mental model of how software actually runs on modern hardware.

The central idea is simple: **algorithmic complexity is only one part of performance.** A program also depends heavily on how data is stored, how often memory is accessed, whether those accesses are predictable, and whether the CPU can keep the required data close to itself.

## 1. Why Do CPU Cache and Hardware Prefetching Matter?

Modern CPUs are much faster than main memory. If the processor had to wait for RAM on every load, much of its computing capacity would be wasted.

To reduce this gap, processors use several levels of cache:

```text
CPU
 ↓
L1 Cache
 ↓
L2 Cache
 ↓
L3 Cache
 ↓
RAM
```

The closer the data is to the CPU, the cheaper it is to access.

Memory is also usually transferred in blocks called **cache lines**, rather than one variable at a time. This means that when the CPU requests one value, nearby values are often loaded as well.

This makes sequential access particularly efficient:

```text
a[0] → a[1] → a[2] → a[3] → a[4]
```

The processor can detect this pattern and use **hardware prefetching** to load future data before the program explicitly requests it.

By contrast, an irregular pattern such as:

```text
a[20] → a[7000] → a[31] → a[90000]
```

is harder to predict and more likely to cause cache misses.

The important principle is therefore:

> Predictable and local memory access allows the CPU to hide memory latency.

---

## 2. How Should I Reason About the Speed of a Program?

Big-O complexity tells us how the amount of work grows as the problem becomes larger, but it does not tell us the full cost of running that work on real hardware.

Two programs can both be \(O(N)\) while having very different execution times.

For example:

```cpp
for (int i = 0; i < n; i++)
    sum += array[i];
```

and:

```cpp
while (node) {
    sum += node->value;
    node = node->next;
}
```

are both linear.

However, the first scans contiguous memory, while the second may repeatedly follow pointers to unrelated memory locations.

The first version benefits from:

- good cache locality,
- hardware prefetching,
- predictable memory access.

The linked structure may suffer from:

- cache misses,
- pointer chasing,
- dependent memory loads.

So a better mental model of runtime is:

\[
\text{Runtime}
\approx
\text{amount of computation}
+
\text{memory traffic}
+
\text{cache misses}
+
\text{dependencies and branches}
\]

Big-O remains essential, especially when comparing polynomial and exponential algorithms, but once the high-level algorithm is reasonable, memory behavior can become a major determinant of actual performance.

---

## 3. What Does It Mean to “Reason About Memory”?

Reasoning about memory means asking more than how much RAM the machine has.

For any important data structure, I should ask:

- How many elements are there?
- How large is each element?
- Which fields does the algorithm actually use?
- Which fields are usually accessed together?
- Is the data contiguous or scattered?
- Is access sequential or random?
- Are there many pointers?
- Are there many heap allocations?

Suppose there are one million objects and each object occupies 128 bytes:

\[
1,000,000 \times 128B \approx 128MB
\]

If an important loop only needs 8 bytes from each object, then most of the data moved through the cache may be irrelevant to that computation.

So memory reasoning is fundamentally about:

> How many useful bytes does the CPU need, and how much unnecessary data must it move to get them?

---

## 4. Why Use Structure of Arrays Instead of Array of Structs?

Consider this representation:

```cpp
struct Module {
    int id;
    float power;
    bool active;
};

Module modules[N];
```

This is an **Array of Structs (AoS)**.

In memory, the layout is roughly:

```text
[id power active][id power active][id power active]...
```

Suppose an algorithm only needs to process `power`.

The processor still loads cache lines containing `id` and `active`, even though those fields are not needed.

A **Structure of Arrays (SoA)** representation would instead be:

```cpp
int ids[N];
float powers[N];
bool active[N];
```

Now the power values are stored together:

```text
30 60 30 90 60 30 60 ...
```

A loop that only processes power can use its cache lines much more efficiently.

The correct lesson, however, is not:

> SoA is always better.

The deeper principle is:

> Data that is frequently used together should usually be stored together.

If every operation needs `id`, `power`, and `active` simultaneously, an AoS layout may be perfectly reasonable.

Data-oriented design therefore starts from the **access pattern**, not from a fixed preference for one representation.

---

## 5. What Is the Problem With Traditional Object-Oriented Design?

Object-oriented programming is not inherently wrong.

The performance criticism is that traditional OOP can encourage programs to organize data according to conceptual objects rather than according to how computations actually consume data.

For example:

```cpp
class Plug {
    Contactor* contactor;
};

class Contactor {
    Bus* bus;
};

class Bus {
    Module* module;
};
```

Conceptually this is very natural:

```text
Plug → Contactor → Bus → Module
```

But these objects may be located far apart in memory.

The processor may have to execute:

```text
load Plug
↓
read pointer
↓
load Contactor
↓
read pointer
↓
load Bus
↓
read pointer
↓
load Module
```

This is called **pointer chasing**.

The important problem is that the address of the next load may depend on the result of the previous load.

That creates three performance issues:

1. **Scattered memory**  
   Objects may occupy unrelated cache lines.

2. **Cache misses**  
   Each pointer may lead to data that is not already cached.

3. **Dependent loads**  
   The processor cannot know where to load the next object until the previous pointer has been read.

This also makes hardware prefetching more difficult.

Data-oriented design therefore changes the starting question.

Traditional object-oriented thinking often begins with:

> What objects exist in the problem domain?

Data-oriented thinking begins with:

> What data does this computation need, and in what order will it access that data?

The broader industry has not simply “abandoned OOP.” Rather, performance-sensitive areas such as game engines, databases, compilers, simulation systems, and high-performance computing increasingly combine higher-level abstractions with more data-oriented internal representations.

---

## 6. Why Are Some Systems Moving Away From Exceptions?

The same hardware-aware reasoning also applies to error handling.

An ordinary explicit error check may look like:

```cpp
result = parse(input);

if (!result)
    handle_error();
```

The program performs a normal return, a check, and a branch.

By contrast, when a C++ exception is actually thrown, the runtime may need to:

```text
find an exception handler
↓
unwind the stack
↓
destroy local objects
↓
transfer control to catch
```

This can be much more expensive than checking an ordinary result value.

The exact claim that one mechanism costs 7 cycles while another costs 5,000–10,000 cycles should not be treated as a universal constant. The cost depends on the compiler, platform, stack depth, and implementation.

The useful principle is simpler:

> Exceptions can be appropriate for genuinely exceptional failures, but they are often a poor mechanism for frequent and expected control flow.

Languages and modern APIs increasingly expose expected failure directly in the type system.

For example, Zig uses error unions conceptually similar to:

```text
Error or Value
```

Other languages and libraries provide:

```text
Optional<Value>
Result<Value, Error>
nullable values
```

This makes ordinary failure explicit and allows it to be handled using predictable control flow.

The distinction is therefore:

```text
Expected failure
→ explicit result / optional / error value

Rare exceptional failure
→ exception may be reasonable
```

---

## Putting the Ideas Together

Most of these questions reduce to one larger model:

```text
CPU is very fast
        ↓
RAM is relatively slow
        ↓
Caches reduce the gap
        ↓
Memory access patterns matter
        ↓
Sequential + predictable access
        ↓
locality + prefetching
        ↓
efficient execution
```

The opposite pattern is:

```text
large scattered objects
        ↓
pointers
        ↓
pointer chasing
        ↓
cache misses
        ↓
dependent and unpredictable memory loads
        ↓
CPU waits for data
```

This leads to the central idea of Data-Oriented Design:

> A program is ultimately a transformation of data. Good design therefore requires understanding not only the conceptual structure of the problem, but also what data is processed, how much of it exists, how it is laid out in memory, and how the processor will access it.

For algorithm design, the first question is still whether the algorithm scales at all. An exponential algorithm does not become scalable simply because its memory layout is efficient.

But once the algorithmic structure is reasonable, understanding caches, memory layout, access patterns, pointers, and control flow provides the next level of reasoning needed to explain why one implementation can be dramatically faster than another.

