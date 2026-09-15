# Lab 3 Exercise

*Authors: Daniel Zhang, Jake Tuero*

> [!WARNING]
> Due: September 15th 2026, 11:30pm

> [!IMPORTANT]
> <RepoCard repo="UofA-CMPUT350/lab-3-exercise"></RepoCard>
> Click `Use this template` button to create your repo based on it

## Rules

- You are allowed to use any of the course material, including this lab's prep material, solution, as well as our C++
  notes.
- You must check for the appropriate preconditions/postconditions. Your program shouldn't crash or have undefined
  behaviour (**hint**: use asserts)!
- Make sure to test **all** implemented code. Use your compiler's address sanitizer (`g++` flags `-fsanitize=address`),
  or
  tools like `valgrind` and `leaks` to check for memory leaks. Your code should compile without errors and without
  warnings with the command
  ```bash
  cmake --preset debug
  cmake --build build
  ```
- In case your program hangs, use ctrl-c to terminate it.
- Remember that you need to include the appropriate header files. To find out which ones you need for specific functions
  such as printf, use the man command.

## Background

### Shared ownership

A *shared pointer* is a smart pointer which implements *shared ownership*. When ownership of a resource is shared by
multiple owners, it is freed if and only if the **last surviving owner is destructed**. In this exercise, we will
implement a class template `SharedPtr<T>`.

There are multiple ways to implement a shared pointer. For example, those familiar with Rust may be aware that Rust's
`Arc` is implemented with 8 bytes (on 64-bit machines). C++’s shared pointer must support what's known as the *aliasing
constructor*, and hence requires 16 bytes. We will follow the C++ convention here.

A **shared pointer** stores two raw pointers:

1. a pointer to a heap-allocated *control block*, and
2. the raw pointer the shared pointer is meant to represent, when you dereference it ("*stored pointer*").

For simplicity, in this exercise we will ignore weak pointers, custom deleters, custom allocators, and thread-safety.
Furthermore, you may assume that `T` is not an array type, nor a reference type. In this simplified model, the **control
block** stores two things:

1. A *reference count* (“refcount”), the number of shared pointers which own the underlying resource.
2. One of (but not both simultaneously):
    1. a pointer to the resource being owned/managed, or,
    2. **(Bonus)** the resource being owned/managed itself, embedded inside the control block.

The pointer-to-resource case occurs when we construct a `SharedPtr<T>` by passing it a raw `T*` to take ownership of.
The embedded-resource case occurs when we construct a `SharedPtr<T>` directly with `makeShared<T>`.

::: important

The resource being *owned/managed* in the control block is usually, but not necessarily always, the same pointer the
`SharedPtr<T>` points to in its raw pointer field (the “stored pointer”).

:::

The semantics (and implementation) of a `SharedPtr<T>` are as follows:

- *Initialization*: A new `SharedPtr<T>` constructed with no arguments just stores `nullptr` for its stored pointer and
  control block pointer. A new `SharedPtr<T>` constructed with a `T*` should allocate a control block (on the heap)
  which stores its resource. The refcount begins at 1, and we begin as the sole owner of the resource. The
  `SharedPtr<T>` will also set a “stored pointer”, which may not be the same as the managed pointer (if created with the
  aliasing constructor (bonus)).
- *Destruction*: Upon destruction, a `SharedPtr<T>` should decrease the refcount of its control block (if it exists) by
    1. If the refcount ever hits 0, the destructor is also responsible for deleting the control block, whose destructor
       should then in turn destroy and deallocate the managed `T`.
- *Copy semantics*: A copy of a `SharedPtr<T>` shares the same “stored pointer” *and* the same control block as the
  original (i.e., implying shared ownership of the same resource). A copy should increase the refcount by 1 (if it
  exists). If the contents were copied into an existing `SharedPtr<T>`, destruction semantics should be invoked on the
  previously managed resource somehow.
- *Move semantics*: A move of a `SharedPtr` should steal the “stored pointer” *and* the control block of the original,
  not modifying the refcount. If the contents were moved into an existing `SharedPtr<T>`, destruction semantics should
  be invoked on the previously managed resource somehow.

### Aliasing

In this exercise, implementing the *aliasing constructor* is a bonus problem. However, even if you don't want to attempt
it, it's good to know why it exists (why C++'s shared pointer has both a managed pointer in the control block, and a
*stored pointer* in the shared pointer itself). Otherwise, you may be confused during implementation why you have two of
the same pointer when you don't do aliasing.

```cpp
struct BigObject {};
struct BigObjectManager {
    BigObject bigObject;
    // Other stuff
};

// What happens if we don't have aliasing
SharedPtr<BigObjectManager> ptrManager = makeShared<BigObjectManager>();
// Lets make a pointer to the big object. Since these are expensive to make,
//   lets just point to the sub-object stored by ptrManager
SharedPtr<BigObject> ptrObject(&ptrManager->bigObject);
// ptrManager now gets freed
ptrManager.reset();
// UH OH, we are trying to delete the object pointed by ptrManager.
// Since ptrObject relies on that same object, its now a dangling pointer!


// With aliasing:
SharedPtr<BigObjectManager> ptrManager = makeShared<BigObjectManager>();
// ref count of object pointed by ptrManager is 1

// Lets make a pointer to the big object.
SharedPtr<BigObject> ptrObject(ptrManager, &ptrManager->bigObject);
// ref count of object pointed by ptrManager is 2

ptrManager.reset();
// The BigObjectManager object still exists (ref count is now 1),
//   so the pointer stored by ptrObject to the BigObject is still valid!
```

## Problems

Write all of your implementations in `SharedPtr.h` and your tests in `main.cpp`. Make sure to test **all** implemented
code. Use your compiler's address sanitizer (g++ flags `-fsanitize=address`), or tools like `valgrind` and `leaks` to
check for memory leaks. Your code should compile without errors and without warnings with the command

```bash
cmake --preset debug
cmake --build build
```

Assert all preconditions with `assert()` from the `<cassert>` header.

### Control Block Base Class

Here is a skeleton for an [abstract base class](../../misc/cpp.md#abstract-classes) for our control block. It has a pure
virtual function, `virtual void* managedAddress()`, which just returns the address of the `T` being managed (the derived
classes will be responsible for implementing this). Since any control block will always need to maintain a refcount, it
is ideal to implement refcounting directly in this base class.

```cpp
class ControlBlockBase {
public:
    ControlBlockBase(); // TODO: implement the default constructor.

    // dtor is virtual, so that we can call derived class's dtor from a ptr to this base class.
    virtual ~ControlBlockBase(); // TODO: implement the destructor.

    // pure virtual function; must be overriden by derived classes
    virtual void* managedAddress() = 0;

    // Delete copies, which also implicitly deletes moves.
    ControlBlockBase(const ControlBlockBase&) = delete;
    ControlBlockBase& operator=(const ControlBlockBase&) = delete;

    long increment()
    {
        // TODO: increment refcount by 1 and return result.
    }

    long decrement()
    {
        // TODO: decrement refcount by 1 and return result.
    }

    long refCount() const
    {
        // TODO: just return the refcount.
    }

private:
    // TODO: add field(s) which both control block types need to have
};
```

Why do we need this? To support two versions of the control block; the one which stores a pointer `T*`, and the
**bonus** which embeds the `T` directly.

Copy and paste the above code snippet into your `SharedPtr.h` and complete the TODOs, namely, determine any data fields,
and implement the ctor, dtor, `increment`, `decrement`, and `refCount`. **Assert any preconditions!**

### Control Block

Publicly derive[+note-1] from `ControlBlockBase` a new concrete class template `ControlBlock<T>`. This concrete control
block implementation will inherit your refcounting functionality from `ControlBlockBase`, and additionally store a
(`private`) `T*`, pointing to the managed object. You should additionally implement:

[+note-1]: In case you forget the syntax for this, refer to our C++ notes [Inheritance access specifiers](../../misc/cpp.md#inheritance-access-specifiers).

- A constructor which takes a `T*` (pointer to resource) to manage,
- A overriding destructor which deletes its managed pointer
- An overriding implementation of `void* managedAddress()` which returns the address of the managed resource.

### Shared Pointer

Implement the class template `SharedPtr<T>`. Refer to the semantics/implementation details of `SharedPtr<T>` described
in [Background](#background). In particular, implement the following, and **determine whether they should be
`const` methods** (we have not indicated for you):

- Two private pointer fields, one for the “stored pointer”, and one pointing to `ControlBlockBase` (**NOT**
  `ControlBlock<T>`).
- A default constructor which sets both its stored pointer and its control block pointer to `nullptr`.
- A constructor which takes a `T*` and both uses it as the “stored pointer”, and takes ownership of it in a new
  `ControlBlock<T>`.  
  *Note*: the presence of these two constructors means it is possible for the control block pointer to be `nullptr`, or,
  for it to not be `nullptr` but for the actually managed pointer in the control block to be `nullptr`.
- A destructor.
- Copy and move constructors.
- Copy and move assignment operators.
- A dereference operator `T& operator*()`, which returns a reference to the **stored**[+note-2] object, and allows you
  to dereference the stored pointer like so: `*mySharedPtr`.
- An arrow operator `T* operator->()`, which allows you to access members of the stored pointer like so:
  `mySharedPtr->methodOfT()`, `mySharedPtr->fieldOfT`. This just returns the stored pointer itself. Consider that we
  will always use this method in conjunction with accessing members.
- `T* get()`, which just returns the stored pointer.
- A equality comparison operator `bool operator==(const SharedPtr<T>& other)`, which compares the stored pointers. This
  automatically gives you for free `operator!=(const SharedPtr<T>&)`.
- A boolean conversion operator, which returns `true` iff the stored pointer is not `nullptr`.
- `void swap(SharedPtr<T>& other)`, which should swap both the stored pointer and the managed control block pointer with
  `other`. Think about what should happen with the refcounts.
- `void reset()`, which releases ownership of the managed object (i.e., invoking the destruction semantics described in
  the Background) and also sets the stored pointer to `nullptr`.
    - An overload, `void reset(T* other)`, which releases ownership of the managed object (again invoking destruction
      semantics), and begins both managing *and* storing `other`. This will require creating a new control block.  
      This must support self-assignment (i.e., `sharedPtr.reset(sharedPtr.get())`) without accidental deletion.  
      **BIG Hint**: It is possible to implement both versions of `reset` using only the constructor and `swap`, as
      one-liners. This is not strictly required by us, however.
- `long useCount()`, which just returns the refcount of the currently managed object.
- **Bonus: Aliasing constructor (+5%).** Implement the *aliasing constructor* template for `SharedPtr<T>`:
  `template <typename U> SharedPtr(const SharedPtr<U>& other, T* storedPtr)`
  which does copy semantics on the same managed resource as `other` (i.e., sharing control block and increasing
  refcount), but uses `storedPtr` as our stored pointer instead.

[+note-2]: NOT the managed object!

It is also to your benefit to know that the STL shared pointer also has template copy- and move- constructors which can
accept `SharedPtr<U>`'s where `U` is *implicitly convertible to* `T`, allowing you to “convert” shared pointer types.
However, don't worry about implementing these today.

### Creating a Shared Pointer

Outside your class, implement the following function template:  
`template <typename T, typename... Args> SharedPtr<T> makeSharedBasic(Args&&... args)`

which perfectly forwards `args...` as arguments to the constructor of a new heap-allocated `T`, and then returns a new
`SharedPtr<T>` which manages (and stores) that raw pointer.

### Written Questions

In `lab3.txt`, answer the following questions in 1-2 sentences each **(focus on what is most important)**:
1. What is a benefit (besides aliasing) of our `SharedPtr<T>` directly storing a “stored pointer” (we could always just
   access it through the control block, if we're not aliasing)?
2. What is a disadvantage of having the stored pointer (instead of going through the control block)?
3. From a system design/architecture perspective, what disadvantages are there to using shared pointers?
4. From a performance perspective, what disadvantages there are to using shared pointers (say, versus a unique pointer
   or raw pointer)?

### Bonus: Embedded Control Block (+15%)

Implement another version of the control block which embeds a `T` directly inside of it, together with the true
`makeShared<T, Args...>`. Write a class template `ControlBlockEmbedded<T>` which publicly inherits from
`ControlBlockBase`. It will need:

- A private field of type `T`, instead of `T*`.
- A constructor template which takes a variadic list of forwarding references `Args&&... args` and perfectly forwards
  them as constructor arguments to initialize its field of type `T`.
- A destructor, and
- An overriding implementation of `void* managedAddress()`.

Once that is done, write an `explicit` constructor[+note-3] for `SharedPtr<T>` that directly takes a stored pointer and
a pointer to a `ControlBlockBase`, and just directly sets them as its members. You can assume all calls to this
constructor are the first-time use of the given control block.

- For a deduction on the bonus, make this constructor a `public` member of `SharedPtr<T>`.
- For the full bonus, make this constructor a `private` member.

[+note-3]: Just write the constructor as you normally do, with the keyword `explicit ` before it. Prevents implicit conversions.

Finally, write  
`template <typename T, typename... Args> SharedPtr<T> makeShared(Args&&... args)`

which directly perfectly forwards `args...` to a new heap-allocated instance of a `ControlBlockEmbedded<T>`, and then
returns a `SharedPtr<T>` constructed using the explicit constructor, passing a pointer to the embedded `T` as the stored
pointer and the control block itself. A cast may be needed somewhere here.

If you chose the full bonus option above, you will need to make
`template <typename T, typename... Args> SharedPtr<T> makeShared(Args&&... args)` a `friend` of `SharedPtr<T>`. As a
hint, to make a function template `template <typename U, typename V> U f(V v)` a `friend`:  
`template <typename U, typename V> friend U f(V v);`
