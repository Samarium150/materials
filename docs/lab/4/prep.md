# Lab 4 Prep Problems

*Author: Daniel Zhang*

> [!WARNING]
> Due: September 22nd 2026, 2:00pm

> [!IMPORTANT]
> <RepoCard repo="UofA-CMPUT350/lab-4-prep"></RepoCard>
> Click `Use this template` button (NOT `fork`) to create your repo based on it

> [!WARNING]
> Do not modify the provided `debug` preset in `CMakePresets.json`,
> as it may cause CI (GitHub Actions) failure.
> Add your own preset instead if you don't want to use the provided one.

## Timing

Writing performant code requires understanding and measuring the time costs of different implementation choices.

In Lab 2, we [introduced](../2/sfml.md#clocks-and-time) `sf::Clock` and `sf::Time`, which were implemented before a
granular enough timing facility was standardized in the standard library. In this lab, we see how to do timing without
drawing upon the heavy dependency of SFML.

Since C++11, the standard library has provided portable (*read: platform-agnostic*) time tracking utilities via the
`<chrono>` header. We can time the wall-clock execution time of a block of code via:

```cpp
#include <chrono>
#include <iostream> // std::cout

void measure()
{
    std::chrono::time_point<std::chrono::steady_clock> t1 = std::chrono::steady_clock::now();
    // ... some code to measure here ...
    std::chrono::time_point<std::chrono::steady_clock> t2 = std::chrono::steady_clock::now();
    std::cout << "Code took "
              << std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count()
              << " milliseconds to execute\n";
}
```

It isn't important to understand all the details at the moment, but for those interested, language-level details are
in the [Timer Details appendix](#appendix-timer-details).

The standard library provides some helper instantiations of the `std::chrono::duration<Rep, Period>` type template,
which correspond to the time units you'd expect:

- `std::chrono::nanoseconds`
- `std::chrono::microseconds`
- `std::chrono::milliseconds`
- `std::chrono::seconds`
- `std::chrono::minutes`
- `std::chrono::hours`

and so on (see the [helper types](https://en.cppreference.com/w/cpp/chrono/duration.html#Helper_types)). We can replace
`std::chrono::milliseconds` in the instantiation of the function template `std::chrono::duration_cast` with any of
these durations to obtain the time taken in different units.

Since the return type of `std::chrono::steady_clock::now()` is well-defined and known at compile time, your compiler
can automatically infer the types of `t1` and `t2`. Thus we can use `auto` to make our code more readable:

```cpp
auto t1 = std::chrono::steady_clock::now();
// ... code to be measured here ...
auto t2 = std::chrono::steady_clock::now();
auto microsecondsElapsed = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count();
```

## Pseudo-random Number Generators (Primer)

A pseudo-random number generator is an algorithm which produces a sequence of numbers which appear random, but are
actually deterministically produced given a (typically relatively small) state. The quality of a PRNG is determined
by factors such as the apparent statistical independence of each element, the length of the *period* (i.e., the length
of the subsequence before you return to the same state and hence cycle the same numbers), and its speed of execution.

The internal state of a PRNG is typically set by setting a *seed*.

In C++, number generators are typically defined as *functors*, i.e., classes which have an `operator()()` defined such
that you can call the object as if it were a function.

Maybe the most commonly used PRNG in C++ is the Mersenne Twister, which can be found in the `<random>` header as
`std::mt19937` for an unsigned 32-bit generator, and as `std::mt19937_64` for the unsigned 64-bit generator. We can use
it like this:

```cpp
#include <random> // mt19937_64
#include <cstdint> // uint64_t

// ... in some function
std::mt19937_64 rng(0); // initialize with seed of 0
uint64_t next = rng(); // next is now the next generated random number
// ... do some stuff
next = rng(); // now next is a different random number.
```

The Mersenne Twister has a very large period of length $2^{19937} - 1$, and passes most statistical tests. However, it
still fails some statistical tests, uses 19937 bits (that's almost 2.5 KiB of memory) for its state (!), and is not
particularly fast.

If you are interested, the permuted congruential generator (PCG)
([Wikipedia](https://en.wikipedia.org/wiki/Permuted_congruential_generator)) is a PRNG which is trivial to implement,
requires only 8–16 bytes of state, is faster, and has better statistical properties than the Mersenne Twister, but it is
not yet in the standard library. As an easy exercise, feel free to implement PCG as a functor.

For the purposes of the problems below, we will assume you use the 64-bit Mersenne Twister as provided by the standard
library, but you may of course also use PCG.

## Type Aliasing

You may recall from C that you can alias types using the `typedef` keyword, like `typedef int* IntPointer;`, which now
means `IntPointer` can be used instead of `int*`. In C++, this is still possible, but since C++11, the `using` keyword
can be used too:

```cpp
using IntPointer = int*;
```

An advantage of `using` is that it can be templated:

```cpp
template <typename T>
using Pointer = T*; // Pointer<T> is now an alias for T*.
```

## Problems

### Timer

We'd like to be able to conveniently reuse the above code pattern, without copying and pasting it all over the place.
A `Timer` is always running once constructed, and it allows the user to get the amount of time that has passed, in
user-designated units, since the `Timer` was either last clicked (or since construction, for the first click).
In `timer.h`, implement `Timer`.

The `Timer` class should have types in its namespace corresponding to nanoseconds, microseconds, milliseconds,
seconds, minutes, and hours. Inside the class, alias the following names to the standard library duration equivalents:

- `Timer::Nanos`
- `Timer::Micros`
- `Timer::Millis`
- `Timer::Seconds`
- `Timer::Minutes`
- `Timer::Hours`

Implement the following methods. **We have not indicated which should be `const`:**

- A default constructor.
- `void restart()`, which resets the state of the `Timer` as if it has only started timing now.
- A function template `template <typename T> uint64_t click()`, whose type parameter takes one of the type aliases you
  defined earlier. For example, calling `click<Timer::Micros>()` returns the number of microseconds since the `Timer`
  was last clicked/restarted/constructed, whichever has happened most recently. Also note that `uint64_t` is given in
  the `<cstdint>` header.
- A function template `template <typename T> uint64_t glance()`, whose type parameter takes one of the type aliases you
  defined earlier. For example, calling `glance<Timer::Micros>()` returns the number of microseconds since the `Timer`
  was last clicked/restarted/constructed, whichever has happened most recently.

Notice that we asked you to take the time unit as a template type parameter. For your own understanding, consider:
what if we instead passed it as an `enum` (e.g., `TimeUnit::MICROS`) function parameter? How would the implementation of
`click` change? Would it be easier/harder to maintain? Would it be more/less efficient?

### Linked Lists vs Contiguous Arrays

The C++ STL provides a heap-allocated dynamic array of `T`s as `std::vector<T>` in `<vector>`
([documentation](https://en.cppreference.com/w/cpp/container/vector.html)). Furthermore, it provides a doubly-linked
list implementation as `std::list<T>` in `<list>`
([documentation](https://en.cppreference.com/w/cpp/container/list.html)). Both allow you to append elements to the end
via `push_back`.

**Problem.** In `list_vs_array.cpp`, use your `Timer` class to time how long it takes to insert 16,000,000 random
`uint64_t`s into a `std::list<uint64_t>`, and then 16,000,000 random `uint64_t`s into a `std::vector<uint64_t>` as
generated by your PRNG of choice. You should test the `std::vector` case both with `reserve` and without. Seed your
PRNG with 0 before starting to fill each container, so that they should both contain exactly the same numbers. You can
explicitly re-seed a `std::mt19937_64` via a call to its `seed` method. Finally, print out the time each case took, in
microseconds.

Next, iterate through each container (if you filled the `std::list` first, do it first again here) and time how long it
takes to sum the elements into a `uint64_t`. This will of course overflow, but that is not important. Do the same for
the `std::vector`. Print the sum and then the time in microseconds both took.

Compile with `cmake --build release`, and also run without optimizations (i.e., with `cmake --build debug`) to give
yourself some intuition on the difference.

::: info Questions to Consider

- How do the results for each container compare? Why?
- What was the performance difference between reserving and not reserving for the `std::vector`? Consider how this
  might change if we used a type that has special copy semantics and no move semantics, instead of a trivially
  copyable type.
- Suppose you have a very nice desktop processor (say,
  a [Ryzen 9 9950X3D2](https://www.amd.com/en/products/processors/desktops/ryzen/9000-series/amd-ryzen-9-9950x3d2-dual-edition.html))
  that has an L3 cache size of 192 MB. Suppose the cache uses an LRU [+lru] eviction policy. Each `uint64_t` is 8 bytes. 
  Why did we choose the container size of 16,000,000 for our benchmark? What might happen if the containers were 
  significantly smaller?
:::

[+lru]: LRU means “Least Recently Used”: when you add an entry into a full cache, the *least recently used entry* is
  removed from the cache.

### Bit Manipulation

We define for $\mathrm{scale} \geq 1$ the function

```cpp
uint64_t expand(uint64_t input, uint32_t scale)
```

which takes the binary representation of `input`, and inserts $(\mathrm{scale} - 1)$ zero bits to the left of each
original bit. For example,

$$
\mathrm{expand} (0b1111, 3) = 0b001001001001
$$

and

$$
\mathrm{expand} (0b0101, 2) = 0b00010001
$$

effectively expanding each original bit to occupy $\mathrm{scale}$ bits. In C++, prepending `0b` to an integer literal
indicates it is represented in binary, and prepending `0x` indicates a hexadecimal literal. In the case that the
expansion causes the result to be larger than 64 bits, we truncate the leftmost (most significant) bits.

**Problem.** In `expand.cpp`, implement `expand` with no more than $O (\text{word size})$ arithmetic or bitwise
operations, where the word size is the number of bits the processor can store in a single register to represent a
memory address. In `main()`, test `expand` to ensure it is correct. The program should compile without errors nor
warnings with `cmake --build debug`. Use `assert()` from the `<cassert>` header to assert all preconditions.

::: warning
It is undefined behaviour to left-shift a 64-bit unsigned integer by 64 or more.
:::

::: info Word Size
Modern architectures are typically fixed at 64 bits, but the idea is that if we were to implement `expand` for a
128-bit architecture, we would implement it for a `uint128_t input` instead. In this case, the big-$O$ bound implies it
would take about double the arithmetic/bitwise operations (which now hypothetically operate on 128-bit words in
constant time) to execute compared to 64 bits.
:::

## Appendix: Timer Details

```cpp
#include <chrono>
#include <iostream>

void measure()
{
    std::chrono::time_point<std::chrono::steady_clock> t1 = std::chrono::steady_clock::now();
    // ...
    std::chrono::time_point<std::chrono::steady_clock> t2 = std::chrono::steady_clock::now();
    std::cout << "Code took "
              << std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count()
              << " milliseconds to execute\n";
}
```

- `std::chrono::steady_clock` is a class implementing a monotonic clock. This means that the time points of this clock
  cannot decrease as time moves forward.
- `std::chrono::steady_clock::now()` is a static function of `std::chrono::steady_clock` which returns an object of
  type `std::chrono::time_point<std::chrono::steady_clock>`, representing specific points in time.
- The difference between two time points is a `std::chrono::duration<rep, period>`, representing lengths/durations of
  time between two time points as a number of *ticks*, where:
    - `rep` is a library-chosen arithmetic type indicating the maximum number of clock ticks supported, typically an
      integer type (e.g., `long long`). In the
      [helper types table](https://en.cppreference.com/w/cpp/chrono/duration.html#Helper_types), we see guarantees about
      the number of bits `rep` might store for the common helper duration types.
    - `period` is a `std::ratio<Num, Denom>` indicating the duration of a single tick in seconds. For example,
      `std::milli = std::ratio<1, 1000>`, meaning each tick of a duration using `std::milli` as a `period` is a
      thousandth
      of a second.
- `std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1)` casts the duration `t2 - t1` into an equivalent
  `duration` which now has a `rep` type of at least 45 bits and which uses `std::milli` as the `period` per tick.
- Finally, all instances of `std::chrono::duration` have a `count()` method which gives you the number of ticks stored
  relative to the `duration`'s `period`. `count` returns an instance of type `rep` (here, this is with respect to
  `std::chrono::milliseconds`), which might be aliased to `long long`.
