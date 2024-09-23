# Some Tips about high efficiency hashmap


## folly's AtomicHashMap 

* requires knowing the approximate number of elements up-front and the space for erased elements can never be reclaimed. 

## [parallel-hashmap](https://github.com/greg7mdp/parallel-hashmap)

**header-only**

* Very efficient, significantly faster than your compiler's unordered map/set or Boost's
* [For a full writeup explaining the design and benefits of the Parallel Hashmap](https://greg7mdp.github.io/parallel-hashmap/)

## [abseil flat_hash_map](https://abseil.io/docs/cpp/guides/container)

## [microsoft/L4](https://github.com/microsoft/L4)

* a fixed-size hashtable, where keys and values are arbitrary bytes
* optimized for lookup operations. It uses [Epoch Queue](https://github.com/Microsoft/L4/wiki/Epoch-Queue) (deterministic garbage collector) to achieve lock-free lookup operations.
* supports caching based on memory size and time. It uses [Clock](https://en.wikipedia.org/wiki/Page_replacement_algorithm#Clock) algorithm for an efficient cache eviction.

## Refs

- [Comprehensive C++ Hashmap Benchmarks 2022](https://martin.ankerl.com/2022/08/27/hashmap-bench-01/)
- [Experiences with Concurrent Hash Map Libraries](https://www.reddit.com/r/cpp/comments/mzxjwk/experiences_with_concurrent_hash_map_libraries/)