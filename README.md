# outline
> [four benefits of std::atomic\<T\>](#four-benefits-of-stdatomict)<br>
> [counting ~~semaphore~~ words](#counting-semaphore-words)<br>
> [convert a map -> vector](#convert-a-map---vector)


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

> _"There are nine spaces in this wording and ten words."_

The algorithm stated above works however, there are some corner cases yet
to be resolved - if there happens to be preceding, trailing or 
consecutive duplicate spaces, it will yield a higher value than expected.

A refined version of our algorithm will be, counting the number of unique
spaces - where unique implies ignoring consecutive duplicate spaces, in 
a giving text, and adding one to that only if there are no trailing or 
preceding spaces.

> _" There are thirteen spaces  in  this wording but ten words. "_

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
<br>

# convert a map -> vector

You know what, there is something fascinating about
_vectors_ - I mean a _dynamic array_ or an _array_. I think
they are quite similar to _maps_, at least, in terms of
being _associative containers_. Elements in a `map` are stored 
in _key-value pairs_ and a `vector` could be considered as such.
Every element in a `vector` has an _index_, which is the _key_ 
and the _element_ held in that position, the `value`.

> A `map` in other languages may be referred to as a _dictionary_
> and `vector`, a _random accessible list_ or an _array_. This 
> technique may be applied in other languages as well.

`std::map<key_t, value_t> -> std::vector<value_t>`

When utilising a map, one may consider any data type
for the _keys_ as long as the _hashing function_ knows how
to crunch such types. You are more than welcome to
overload the hashing function in case it does not yet
support your preferred data type. That is to say we are
almost always able to assign any data type as a _key_ for
_maps_. Things get quite a bit on the edge and may require
some carefulness and patience when converting a `map` to 
`vector`.

Remember we mentioned that _index_ is to a `vector` as the 
_key_ is to a `map`? Selecting a _key_type_ for a `map` is 
quite flexible but for a `vector`, the _index_ has to be an 
_unsigned integral_ or _safely convertible_ to an 
_unsigned integral_. Alright, to make this representation, 
one ought to find a safe means to convert the `key_type` to an 
`integral_t`.

- `char -> unsigned_t` doable
- `int -> unsigned_t` doable - watch out for overflow due to negative values
- `bool -> unsigned_t` this is a joke 😄
- `size_t -> unsigned_t` already an integral_t
- `string -> unsigned_t` requires creativity -
could be as simple as calling `std::from_chars` that's if
the string is actually a integral value held as a string
for example, a short id or age or something like that.
- `pair -> unsigned_t` creativity and care required

We've got snippets of code to illustrate how we could
utilise this technique in our codebases or library 
implementations and benchmarks to assure us that our 
effort may pay off.

Below is an implementation of a function writing data into 
a bucket and we use `std::unordered_map<uint32_t, uint32_t>`. 
The one further down switches the data type for the bucket to
`std::vector<uint32_t>`.

```c++
 1  auto fill_bucket_m(std::uint32_t n) {
 2    std::mt19937 generator{std::random_device{}()};
 3    std::uniform_int_distribution<std::uint32_t> distribution(0, n - 1);
 4
 5    std::unordered_map<std::uint32_t, std::uint32_t> bucket;
 6    bucket.reserve(n);
 7    for (; 0 < n; --n)
 8      bucket[distribution(generator)] = n;
 9    return bucket;
10  }
```
_inserting a **n** values into a std::unordered_map_
<br>
```c++
 1  auto fill_bucket_v(std::uint32_t n) {
 2    std::mt19937 generator{std::random_device{}()};
 3    std::uniform_int_distribution<std::uint32_t> distribution(0, n - 1);
 4
 5    std::vector<std::uint32_t> bucket(n);
 6    for (; 0 < n; --n)
 7      bucket[distribution(generator)] = n;
 8    return bucket;
 9  }
```
_inserting a **n** values into a std::vector_


Running `fill_bucket_m(1'000'000)` finishes in _~120.7 ms_ whiles 
`fill_bucket_v(1'000'000)` completes in _~6.79 to ~22 ms_ depending
on which compiler and libraries used to compiler the code. That's at 
best, _17.78x_ faster. Also, if we are able to express the size 
of the bucket as a _constant expression_, we could use `std::array`
as the data structure for our bucket - it may be _25.75x_ faster.

Alongside the speedup, converting `map` to `vector` saves all the storage 
used by the `map` to hold its _keys_ - roughly 
`map_size * sizeof(map::key_type)` bytes, introducing a quieter memory 
footprint - which helps the entire system to function efficiently.

_Michelle D'Souza_ gives more real-world reports on this topic in her talk titled
[Cache Me Maybe: The Performance Secret Every C++ Developer Needs](https://www.youtube.com/watch?v=VhKq0nzPTh0).

In scenarios where we are unable to convert the keys safely into `unsigned` 
values. we may proceed to use a `map`. It is also recommended to consider 
`unordered_map` and to reserve when you have knowledge of what the size 
of the container will be. _Kevin Carpenter_ elaborates on this in his talk 
[O(1) or O(no-no-no): Mastering the unordered_map](https://accuonsea.uk/2026/sessions/o1-or-ono-no-no-mastering-the-unordered_map/).

I do hope you have fun with this update - talk to you later.

[ ↑ back to the top](#outline)

