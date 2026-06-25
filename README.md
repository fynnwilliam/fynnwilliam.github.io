# outline
> [four benefits of std::atomic\<T\>](#four-benefits-of-stdatomict)<br>
> [counting ~~semaphore~~ words](#counting-semaphore-words)<br>
> [convert a map -> vector](#convert-a-map---vector)<br>
> [zigzag conversion - doing more with data](#zigzag-conversion---doing-more-with-data)<br>
> [wait for the signal](#wait-for-the-signal)


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

There is something fascinating about
_vectors_ - I mean a _dynamic array_ or an _array_. 
They are quite similar to _maps_, at least, in terms of
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
`fill_bucket_v(1'000'000)` completes in _~6 ms_ - that's 
_20.11x_ faster. Also, if we are able to express the size 
of the bucket as a _constant expression_, we could use `std::array`
as the data structure for our bucket - it may be _26.35x_ faster.

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
<br>

# zigzag conversion - doing more with data

Alright, so this is a puzzle I found on the net a while ago and 
may come in handy when formatting text 
or creating animations. Ok so we are to rearrange a string say, 
**PAYPALISHIRING** in a zigzag pattern on a given number of rows like 
this:
```
P   A   H   N
A P L S I I G
Y   I   R
```
then read the characters row by row to form a new string - 
**PAHNAPLSIIGYIR**. We will be implementing the _logic_ for making 
this transformation:

`std::string convert(const std::string &s, size_t rows);`

Observing the zigzag pattern above may seam a bit daunting at first. 
However, if we reveal the grid in which that pattern lies and proceed 
to replace each character with its index, things become a bit clearer.

|      |      |      |      |      |      |      |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| P    |      | A    |      | H    |      | N    |
| A    | P    | L    | S    | I    | I    | G    |
| Y    |      | I    |      | R    |      |      |

_Revealing the grid_


|      |      |      |      |      |      |      |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| 0    |      | 4    |      | 8    |      | C    |
| 1    | 3    | 5    | 7    | 9    | B    | D    |
| 2    |      | 6    |      | A    |      |      |

_Displaying character indices_

_We are using hexadecimal values to maintain a fixed width_

Analysing the grid with indices, one may spot a couple of patterns.
For every zigzag pattern, there is a unique index for each character in the 
string. You may also realise a pattern regarding the distance between 
characters on the same row. We shall observe multiple patterns below to get
more data.

Example 1:

input: s = **PAYPALISHIRING**, rows = **3**<br>
output: **PAHNAPLSIIGYIR**

|      |      |      |      |      |      |      |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| P    |      | A    |      | H    |      | N    |
| A    | P    | L    | S    | I    | I    | G    |
| Y    |      | I    |      | R    |      |      |

<br>

|      |      |      |      |      |      |      |  steps |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--    |
| 0    |      | 4    |      | 8    |      | C    | `-> 4` |
| 1    | 3    | 5    | 7    | 9    | B    | D    | `-> 2` |
| 2    |      | 6    |      | A    |      |      | `-> 4` |

----------------------------------------

Example 2:

input: s = **PAYPALISHIRING**, rows = **4**<br>
output: **PINALSIGYAHRPI**

|      |      |      |      |      |      |      |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| P    |      |      | I    |      |      | N    |
| A    |      | L    | S    |      | I    | G    |
| Y    | A    |      | H    | R    |      |      |
| P    |      |      | I    |      |      |      |

<br>

|      |      |      |      |      |      |      |   steps   |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--       |
| 0    |      |      | 6    |      |      | C    | `-> 6`    |
| 1    |      | 5    | 7    |      | B    | D    | `-> 4, 2` |
| 2    | 4    |      | 8    | A    |      |      | `-> 2, 4` |
| 3    |      |      | 9    |      |      |      | `-> 6`    |

----------------------------------------

Example 3:

input: s = **PAYPALISHIRING**, rows = **5**<br>
output: **PHASIYIRPLIGAN**

|      |      |      |      |      |      |
| :--: | :--: | :--: | :--: | :--: | :--: |
| P    |      |      |      | H    |      |
| A    |      |      | S    | I    |      |
| Y    |      | I    |      | R    |      |
| P    | L    |      |      | I    |  G   |
| A    |      |      |      | N    |      |

<br>

|      |      |      |      |      |      |   steps   |
| :--: | :--: | :--: | :--: | :--: | :--: | :--       |
| 0    |      |      |      | 8    |      | `-> 8`    |
| 1    |      |      | 7    | 9    |      | `-> 6, 2` |
| 2    |      | 6    |      | A    |      | `-> 4`    |
| 3    | 5    |      |      | B    |  D   | `-> 2, 6` |
| 4    |      |      |      | C    |      | `-> 8`    |


Further analysis also reveals:
 - the index of the first element on each row, is the row number.
 - indices of subsequent elements for the first and last row are the 
 multiples of `(rows - 1) * 2` - we call this the _initial steps_.
 - future indices on other rows are the multiples of _current row steps_ - `row_i_steps` 
   - `row_i_steps` has an initial value of `2 * current row number`
   - and updated on each iteration by computing 
   `row_i_steps = initial steps - row_i_steps`. 

You may continue exploring as there is more info buried in this pattern. 
We are good for an implementation with the facts above though.

```c++
 1   std::string convert(const std::string &s, size_t rows) {
 2     if (rows == 1zu || rows >= s.size())
 3       return s;
 4
 5     const auto last_row_index = rows - 1zu;
 6     const auto initial_steps = last_row_index * 2zu;
 7     const auto last = &s.back() + 1;
 8
 9     std::string new_s(s.size(), ' ');
10
11     for (size_t i = 0, x = 0; i < rows; ++i) {
12       auto row_i_steps = 2zu * i;
13       for (auto j = s.data() + i; j < last; j += row_i_steps, ++x) {
14         new_s[x] = *j;
15         row_i_steps = i == 0zu || i == last_row_index
16                           ? initial_steps
17                           : initial_steps - row_i_steps;
18       }
19     }
20
21     return new_s;
22   }
```

_above is a fairly efficient implementation of the zigzag conversion in C++_

So we return early, on _line 3_ if conversion will yield same as the input string.
Then from _line 5_ to _line 7_, we initialise our frequently used variables. We 
create a new buffer on _line 9_ to hold the output string. Finally, we copy the
characters row by row utilising the info we gathered from the grid and return to 
the caller.

Please see the _topdown level 1 metrics_ breakdown after benchmarking the transformation of 
a 1,000,000 character array on 10 rows.

```
# 5.1-full on Intel(R) Core(TM) i5-8350U CPU @ 1.70GHz [kblr/skylake]
C0    FE               Frontend_Bound   % Slots                        5.3  < [33.0%]
C0    BAD              Bad_Speculation  % Slots                        0.3  < [33.0%]
C0    BE               Backend_Bound    % Slots                        7.1  < [33.0%]
C0    RET              Retiring         % Slots                       87.4    [33.0%]<==
```

There may be ways to improve the metrics above so I encourage you to give it a try.

Take care folks, talk to you later.

[ ↑ back to the top](#outline)
<br>

# wait for the signal

The C++ Runtime Library has a predefined flow for when
a crash occurs. There are quite a number of signals
and at least, one gets emitted to notify the reader(s).

What we have to do is register our interest in the
signal(s) of interest as well as what task we intend to
perform when the signal occurs. Before we get into
registering our interest, let's have a quick read on
a couple of signals related to crashes.

1. `SIGINT        External interrupt, usually by the user.`
2. `SIGILL        Invalid instruction.`
3. `SIGABRT       Abnormal termination.`
4. `SIGFPE        Erroneous arithmetic operation.`
5. `SIGSEGV       Invalid memory access.`
6. `SIGTERM       Termination request.`

So we have an API from the C++ Standard Library,
`std::signal` that we use to set the error handler
for a specific signal. Below is the full signature of
the function for clarity.

`extern __sighandler_t signal(int __sig, __sighandler_t __handler) noexcept(true)`

where `__sighandler_t` implies `void (*)(int)`

We will utilise the `std::signal` API in our local or test
builds. Our interest lies in analysing uncaught
exceptions and signals that may cause our program to
terminate suddenly. All we need for now, to perform the
analysis is to print out or save the call stack, then
reset the signal on its default route. Resetting the
signal handler to default has a perk to it - based on
our operating system, we could get the core, dumped
automatically for us.

Below is a possible implementation to register our
signals of interest.

On line 3, we write the current call stack to standard
output. Line 4 and 5 are where we reset the signal
handler to its default and raise the signal again. The default
handler usually dumps the core automatically. Then from
line 8 to 13, we assign each signal a new handler - the
one we implemented. See lines 2 to 6 please.

```c++
 1 const auto install_signal_handlers = [] {
 2   static constexpr auto signal_handler = [](int sig) {
 3     std::println("sig code '{}' occured\n{}", sig, std::stacktrace::current());
 4     std::signal(sig, SIG_DFL);
 5     std::raise(sig);
 6   };
 7
 8   std::signal(SIGABRT, signal_handler);
 9   std::signal(SIGSEGV, signal_handler);
10   std::signal(SIGFPE, signal_handler);
11   std::signal(SIGINT, signal_handler);
12   std::signal(SIGILL, signal_handler);
13   std::signal(SIGTERM, signal_handler);
14 };
```

We use the code below to test our implementation.

```c++
int main() {
  install_signal_handlers();

  throw 42;                    // calls std::terminate -> std::abort -> SIGABRT occurs
  // std::abort();             // SIGABRT occurs
  /*int* ptr = nullptr;
  std::println("{}", *ptr);*/  // SIGSEGV occurs
  // int a = 4 / 0;            // SIGFPE occurs
}
```

You may be wondering why will `throw 42` call
`std::terminate` and so on. Because there is no
_catch-block_ to handle the exception - the C++
Runtime Library is designed to call `std::terminate`
in such cases and `std::terminate` also calls
`std::abort` which then makes `SIGABRT` to occur 
using `std::raise(SIGABRT)`.

You may follow this [Godbolt Link](https://godbolt.org/z/d8oGvafEv) to analyse the code.

Thank you. Untill another time, take care.

[ ↑ back to the top](#outline)
