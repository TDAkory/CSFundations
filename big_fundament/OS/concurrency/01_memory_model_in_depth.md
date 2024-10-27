# 深入理解内存屏障

- [C++ 中的 volatile，atomic 及 memory barrier](https://gaomf.cn/2020/09/11/Cpp_Volatile_Atomic_Memory_barrier/)
- [volatile与内存屏障总结](https://zhuanlan.zhihu.com/p/43526907)
- [X86/GCC memory fence的一些见解](https://zhuanlan.zhihu.com/p/41872203)
- [内存一致性模型](https://blog.csdn.net/langren388/article/details/102702638)
- [Intel® 64 Architecture Memory Ordering White Paper](http://www.cs.cmu.edu/~410-f10/doc/Intel_Reordering_318147.pdf)
- [The memory model of arm](https://developer.arm.com/documentation/100941/0101/The-memory-model)

## Compiler Fence

GCC的compiler fence有一个众所周知的写法：

```c
asm volatile("": : :"memory")
```

它只是插入了一个空指令""，什么也没做。关键在最后的"memory" clobber，它告诉编译器：这条指令（其实是空的）可能会读取任何内存地址，也可能会改写任何内存地址。那么编译器会变得保守起来，它会防止这条fence命令上方的内存访问操作移到下方，同时防止下方的操作移到上面，也就是防止了乱序。

这条命令还有另外一个副作用：它会让编译器把所有缓存在寄存器中的内存变量flush到内存中，然后重新从内存中读取这些值。这并不一定是我们想要的结果，比如有些变量只在当前线程中使用，留在寄存器中很好，多了一对写/读内存操作是不必要的开销。

我们可以通过gcc内联汇编命令的input和output操作符明确指定哪些内存操作不能乱序，如这个例子：

```c
WRITE(x)
asm volatile("": "=m"(y) : "m"(x):) // memory fence
READ(y)
```

这里先对变量x进行写操作，后对变量y进行读操作，中间的内联汇编告诉编译器插入一条指令（其实是空的），它可能会读x的内存，会写y的内存，因此编译器不会把这两个操作乱序。这种明确的memory fence的好处是：使编译器尽量少的对其他不相关的变量造成影响，避免了额外开销。

## CPU Fence

X86属于strong memory model，这意味着在大多数情况下cpu会保证内存访问指令有序执行。具体的说，如果对内存读(Load)和写(Store)操作进行两两组合：LoadLoad, LoadStore, **StoreLoad**, StoreStore，只有StoreLoad组合可能乱序，而且Store和Load的内存地址必须是不一样的。

> Intel 64 memory ordering guarantees that for each of the following memory-access instructions, the constituent memory operation appears to execute as a single memory access regardless of memory type:
> 1. Instructions that read or write a single byte.
> 2. Instructions that read or write a word (2 bytes) whose address is aligned on a 2 byte boundary.
> 3. Instructions that read or write a doubleword (4 bytes) whose address is aligned on a 4 byte boundary.
> 4. Instructions that read or write a quadword (8 bytes) whose address is aligned on an 8 byte boundary.
>
> All locked instructions (the implicitly locked `xchg` instruction and other read-modify-write instructions with a lock prefix) are an indivisible and uninterruptible sequence of load(s) followed by store(s) regardless of memory type and alignment.
> 
> Other instructions may be implemented with multiple memory accesses. From a memoryordering point of view, there are no guarantees regarding the relative order in which the constituent memory accesses are made. There is also no guarantee that the constituent operations of a store are executed in the same order as the constituent operations of a load. 


**Intel 64 memory ordering obeys the following principles:**

1. Loads are not reordered with other loads.
2. Stores are not reordered with other stores.
3. Stores are not reordered with older loads.
4. Loads may be reordered with older stores to different locations but not with older stores to the same location.
5. In a multiprocessor system, memory ordering obeys causality (memory ordering respects transitive visibility).
6. In a multiprocessor system, stores to the same location have a total order.
7. In a multiprocessor system, locked instructions have a total order.
8. Loads and stores are not reordered with locked instructions. 

**Armv8-A implements a weakly-ordered memory architecture.**

1. Memory ordering : https://developer.arm.com/documentation/102336/0100/Memory-ordering

## 内存一致性模型

* 顺序存储模型 sequential consistency model
* 完全存储定序 total store order
* 部分存储定序 part store order
* 宽松存储模型 relex memory order