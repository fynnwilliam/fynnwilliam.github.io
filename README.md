# outline
> [four benefits of std::atomic\<T\>](#four-benefits-of-stdatomict)<br>
> [counting ~~semaphore~~ words](#counting-semaphore-words)


# four benefits of std::atomic\<T\>

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

[ ↑ back to the top](#outline)
<br>

# counting ~~semaphore~~ words

Knowing the number of words in a sentence or a given text is a useful
routine for text processing applications. We will do well and
implement an efficient algorithm for checking how many words are present
in a sentence.

Literally, counting how many spaces there are in a sentence and adding one
to that tells how many words make up that sentence.

> _eg: "There are nine spaces in this wording and ten words."_

The algorithm stated above works however, there are some corner cases yet
to be resolved - if there happens to be preceding, trailing or 
consecutive duplicate spaces, it will yield a higher value than expected.

A refined version of our algorithm will be, counting the number of unique
spaces - where unique implies ignoring consecutive duplicate spaces, in 
a giving text, and adding one to that only if there are no trailing or 
preceding spaces.

> _eg: " There are thirteen spaces  in  this wording but ten words. "_

From the example above, despite we having preceding and trailing spaces, 
as well as two consecutive spaces between the words _spaces_ and _in_ and
same between "in" and "this", we are able to accurately tell how many words
are present in the text.

below is our initial implementation of this algorithm:

```c++
1  inline auto word_count(std::string_view text) noexcept {
2    if (text.empty())
3      return 0uz;
4
5    auto spaces = count_unique_spaces(text);
6    return std::isspace(text.back()) == 0 ? ++spaces : spaces;
7  }
```

So we are hiding all the magic in `line 5` and return based on whether we
spotted a trailing space. We are only concerned about trailing spaces on
`line 6` because our _count for unique spaces_ already ignored preceding
spaces.

Since we've encapsulated our main logic into the `count_unique_spaces`
method, we shall focus on that and improve the performance of our
implementation.

```c++
1  inline auto count_unique_spaces(std::string_view text) noexcept {
2    size_t spaces = 0;
3    for (size_t i = 0, j = 1, size = text.size(); j < size; ++i, ++j) {
4      if (std::isspace(text[i]) == 0 && std::isspace(text[j]) != 0)
5        ++spaces;
6    }
7
8    return spaces;
9  }
```

`count_unique_spaces` counts a space when it is preceded by a
printable character. This approach automatically handles preceding
and consecutive duplicate spaces. Below are some possible changes we
may consider to improve the performance of our routine:
 - on `line 3`, we initialise three variables `i, j` and `size` - we
 only need `size`. It will require us to walk the text in reverse.
 - on `line 4`, we first check for a printable character and then
 check for a whitespace character. Since std::isspace returns a
 non-zero(positive) value if the character is a whitespace
 character and zero, for printable characters, we may only compare
 the returned values from both calls.

 ```c++
1   inline auto count_unique_spaces(std::string_view text) noexcept {
2    size_t spaces = 0;
3    for (size_t size = text.size() - 1; size > 1; --size) {
4      if (std::isspace(text[size - 1]) < std::isspace(text[size]))
5        ++spaces;
6    }
7
8    return spaces;
9  }
```
  _above is the implementation of proposed changes - delivers ~2x speedup_

So that will be it for now - until the next read, take care.

[ ↑ back to the top](#outline)
