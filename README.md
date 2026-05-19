# Outline
> [Four Benefits of std::atomic\<T\>](#four-benefits-of-stdatomict)

<br>
<br>
<br>

# Four Benefits of std::atomic\<T\>

### It makes our code portable

Let's use `arm` and `intel` to elaborate this point. `arm` implements a
`relaxed` memory ordering by default whiles `intel` implements sequencial
consistency. The beauty of `std::atomic<T>`, where `T` is the data type,
in this mix, is the default memory ordering on `std::atomic<T>` being 
`std::memory_order_seq_cst`. This implies sequential consistency. So we can
say, just wrapping a data structure in `std::atomic` generates portable code.

### It helps a developer to express their intent or preference on memory ordering

We have four usable memory orderings for `std::atomic<T>`, five if we count
acquire_release as distinct.

- `std::memory_order_acquire` -> recommended for read operations
- `std::memory_order_release` -> recommended for write operations
- `std::memory_order_acq_rel` -> recommended for read_write operations
- `std::memory_order_seq_cst` -> tells the compiler and or CPU to exclude this operation from out-of-order execution
- `std::memory_order_relaxed` -> tells the compiler and or CPU to include this operation in out-of-order execution

### May introduce a performance boost when memory ordering is done right

Although `relaxed > acquire_release > sequential_consistency`, it is critical
to ensure our code is correct and safe on all platforms of interest.

### Provides lock-free access to some data structures without race conditions

You may confirm whether `std::atomic<T>` has lock-free accesses using
`static_assert(std::atomic<T>::is_lock_free())` before shipping the code.
Also, the selected memory ordering can introduce or remove locks during operations.
