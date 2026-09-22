# Lab 4 Challenge Problems

*Author: Daniel Zhang*

> [!WARNING]
> Due: September 27nd 2026, 11:30pm

> [!IMPORTANT]
> <RepoCard repo="UofA-CMPUT350/lab-4-challenge"></RepoCard>
> Click `Use this template` button (NOT `fork`) to create your repo based on it

> [!WARNING]
> Do not modify the provided `debug` preset in `CMakePresets.json`,
> as it may cause CI (GitHub Actions) failure.
> Add your own preset instead if you don't want to use the provided one.

::: important Optional bonus problems

These problems are optional and worth bonus marks. Working code alone is not sufficient: you must be able to orally
explain it and convince your TA that you understand the solution.

:::

## Problems

### C1. Expand by a Factor of Three

**Optional.** Implement

```cpp
uint64_t expand3(uint64_t input)
```

which is the same as $\text{expand} (\cdot, 3)$ [+expand-three] from
the [bit manipulation prep](prep.md#bit-manipulation),
but with $O (\log (\text{word size}))$ arithmetic or bitwise operations and at most $O (\log (\text{word size}))$ words
of storage.

[+expand-three]: $\text{expand} (\cdot, 3)$ means the function in one variable defined by always plugging in $3$ to the
  second parameter of $\text{expand} (\text{input}, \text{scale})$.

*A concise, clear explanation of what your code does and why it works must be included. In particular, directly explain
and reason about any constants and data transformations. Include a high-level statement of the top-level sequence of
transformations.*

::: tip Hint

Here is a near-solution that, **once fixed**, would be worth full marks (besides the explanation) if the question were
for 8 bits instead of 64:

```cpp
uint8_t expand3(uint8_t input)
{
    input = (input | (input << 4)) & 195;
    input = (input | (input << 2)) & 73;
    return input;
}
```

:::

### C2. Arbitrary Expansion

**Optional.** Implement

```cpp
uint64_t expand(uint64_t input, uint32_t scale)
```

once again taking arbitrary input and scale, as you originally implemented, but now with at most
$O (\log^2 (\text{word size}))$ arithmetic or bitwise operations and at most $O (\log (\text{word size}))$ words of
storage.

This bound is not tight. Once you are done, write a function template

```cpp
template <uint32_t Scale>
uint64_t expand(uint64_t input)
```

which optimizes the above to a general solution with $O (\log (\text{word size}))$ arithmetic/bitwise runtime
operations.
You are allowed a one-off compile-time cost of $O (\log^2 (\text{word size}))$ arithmetic/bitwise operations for any
given
`scale`. In particular, you are **not allowed $O (\text{word size})$ at compile time**.

*A concise, clear explanation of what your code does and why it works must be included. In particular, directly explain
and reason about any constants and data transformations. Include a high-level statement of the top-level sequence of
transformations.*

::: tip Hint 1

Generalize your solution from C1. You may wish to try manually deriving the solution for `scale = 4`.

:::

::: tip Hint 2

One way of implementing “next power-of-two”, which gives the next power-of-two $\geq$ any number, in
$O (\log (\text{word size}))$ steps is:

```cpp
uint64_t nextPOT(uint64_t input) {
    input -= 1;
    input |= input >> 32;
    input |= input >> 16;
    input |= input >> 8;
    input |= input >> 4;
    input |= input >> 2;
    input |= input >> 1;
    return input + 1;
}
```

:::
