# [MESI](https://en.wikipedia.org/wiki/MESI_protocol)

> The MESI protocol is an Invalidate-based cache coherence protocol, and is one of the most common protocols that support write-back caches. 

MESI 是指4中状态的首字母。每个Cache line有4个状态，可用2个bit表示，

**Modified (M)**

    The cache line is present only in the current cache, and is dirty - it has been modified (M state) from the value in main memory. The cache is required to write the data back to main memory at some time in the future, before permitting any other read of the (no longer valid) main memory state. The write-back changes the line to the Shared state(S).

**Exclusive (E)**

    The cache line is present only in the current cache, but is clean - it matches main memory. It may be changed to the Shared state at any time, in response to a read request. Alternatively, it may be changed to the Modified state when writing to it.

**Shared (S)**

    Indicates that this cache line may be stored in other caches of the machine and is clean - it matches the main memory. The line may be discarded (changed to the Invalid state) at any time.

**Invalid (I)**

    Indicates that this cache line is invalid (unused).

When the block is marked M (modified) or E (exclusive), the copies of the block in other Caches are marked as I (Invalid).
