# parlayhash : A Header Only Fast Concurrent Hash Table.

A concurrent hash table supporting **wait-free finds** and **lock-free updates**.
It supports the following interface:

- `unordered_map<K,V,Hash=std::hash<K>,Equal=std::equal_to<K>>(n)` :
constructor for table of initial size n (in growable version n can be 0).

- `find(const K&) -> std::optional<V>` : If the key is in the table, returns the value associated
  with it, otherwise returns std::nullopt.

- `insert(const K&, const V&) -> bool` : If the key is in the table, returns false, otherwise inserts the key
with the given value and returns true.

- `remove(const K&) -> bool` : If the key is in the table, removes the
  key-value and returns true, otherwise it returns false.

- `upsert(const K&, (const std::optional<V>&) -> V)) -> bool` : If the
key is in the table with an associated value v then it applies the function (second argument)
to `std::optional<V>(v)`, replaces the current value for the key with the
returned value, and returns false.  Otherwise it applies the
function to std::nullopt and inserts the key into the table with the
returned value, and returns true.   For example using: `[&] (auto x) {return v;}` will just set
the key to have value v whether as was in there or not. 

- `size() -> long` : Returns the size of the table.  Not linearizable
with the other functions but returns correct size if no other
functions are concurrent.  It takes work proportional to the table
size.

The type for keys (K) and values (V) must be copyable, and might be
copied by the hash table even when not being updated (e.g. when
another key in the same bucket is being updated).

A simple example can be found in [examples/example.cpp](examples/example.cpp)
There are two versions:

- `include/hash_nogrow/unordered_map.h` : Does not support growing the number of buckets.  It can grow arbitrarily large but each buckets will become large and the table will be slow.  The number of buckets is specified when the table is constructed.   

- `include/hash_grow/unordered_map.h` : Supports growable hash tables.  The number of buckets increase by a constant factor when any bucket gets too large.   The copying is done incrementally by each update, allowing for a mostly lock-free implementation (allocation of new arrays is necessarily not lock-free since it must go throught the operating system).

There is a `USE_LOCKS` flag at the start of each file that is
commented out by default.  If uncommented, then the implementation
will use locks.  If using locks the function passed to `upsert` will
be run in isolation (i.e., mutually exclusive of any other invocation
of the function by an upssert on the same key).

**Implementation**: Each bucket points to a structure (Node)
containing an array of entries.  Nodes come in varying sizes and on
update the node is copied.  When growing each bucket is copied to k
new buckets and the old bucket is marked as "forwarded".



## Benchmarks

Benchmarks comparing to other hash tables can be found in `benchmarks`.   With `cmake` the following should work:

    git clone https://github.com/cmuparlay/parlayhash.git
    cd parlayhash
    mkdir build
    cd build
    cmake ..
    make -j
    cd benchmarks
    ./hash_grow           // our growable version
    ./hash_nogrow         // our non growable version
    ./tbb_hash            // [tbb concurrent hash table](https://spec.oneapi.io/versions/latest/elements/oneTBB/source/containers/concurrent_unordered_map_cls.html)
    ./libcuckoo           // [libcuckoo's cuckooohash_map](https://github.com/efficient/libcuckoo)
    ./growt               // [growt's concurrent hash table](https://github.com/TooBiased/growt)
    ./folly_hash          // [folly's ConcurrentHashMap](https://github.com/facebook/folly/blob/main/folly/concurrency/ConcurrentHashMap.h)
    ./boost_hash          // [boost's concurrent_flat_map](https://www.boost.org/doc/libs/1_83_0/libs/unordered/doc/html/unordered.html#concurrent)
    ./parallel_hashmap    // [parallel hashmap](https://github.com/greg7mdp/parallel-hashmap)
    ./folly_sharded       // our own sharded version using folly's efficient [non-concurrent F14map](https://github.com/facebook/folly/blob/main/folly/container/F14Map.h)
    ./abseil_sharded      // our own sharded version using folly's efficient [non-concurrent flat_hash_map](https://abseil.io/docs/cpp/guides/container)
    ./std_sharded         // our own sharded version of std::unordered_map
    ...

For some of these you need to have the relevant library installed (e.g., boost, folly, abseil, tbb).

The benchmarks will run by default on the number of hardware threads you have on the machine.
It will run over two data sizes (100K and 10M), two update percents(5% and 50%), and two distritributions (uniform and zipfian=.99).
Performance is reported in millions operations-per-second (mops) for each combination as well as the geometric mean over all combinations.  Options include:

    -n <size>  : just this size
    -u <update percent>   : just this percent
    -z <zipfian parameter>  : just this parameter
    -grow : starts table at size 1 instead of size n
    -verbose : prints out some extra information
    -t <time in seconds>  : length of each trial, default = 1
    -r <num rounds>  : number of rounds for each size/update-percent/zipfian, default = 2
    -p <num threads> 

