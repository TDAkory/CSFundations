# Cache及优化手段

## Read List

- [如何用 Cache 在 C++ 程序中进行系统级优化（一）](https://toutiao.io/posts/60r22r/preview)
- [如何用 Cache 在 C++ 程序中进行系统级优化（二）](https://toutiao.io/posts/1e33zc/preview)
- [内存屏障今生之Store Buffer, Invalid Queue](https://blog.csdn.net/wll1228/article/details/107775976)

## Cache Info

```shell
cat /sys/devices/system/cpu/cpu0/cache/**/size  # 查询cache容量
cat /sys/devices/system/cpu/cpu0/cache/**/coherency_line_size  # 查询cache-line大小
```

```shell
/s/d/s/c/c/c/index0> pwd
/sys/devices/system/cpu/cpu0/cache/index0
/s/d/s/c/c/c/index0> ls -al
total 0
drwxr-xr-x 3 root root    0 Mar 23  2021 .
drwxr-xr-x 7 root root    0 Jul 19 18:33 ..
-r--r--r-- 1 root root 4096 Mar 23  2021 coherency_line_size
-r--r--r-- 1 root root 4096 Nov 30 14:04 id
-r--r--r-- 1 root root 4096 Mar 23  2021 level
-r--r--r-- 1 root root 4096 Mar 23  2021 number_of_sets
-r--r--r-- 1 root root 4096 Nov 30 14:03 physical_line_partition
drwxr-xr-x 2 root root    0 Jul 19 18:33 power
-r--r--r-- 1 root root 4096 Mar 23  2021 shared_cpu_list
-r--r--r-- 1 root root 4096 Mar 23  2021 shared_cpu_map
-r--r--r-- 1 root root 4096 Mar 23  2021 size
-r--r--r-- 1 root root 4096 Mar 23  2021 type
-rw-r--r-- 1 root root 4096 Nov 30 14:04 uevent
-r--r--r-- 1 root root 4096 Mar 23  2021 ways_of_associativity
```

## Store Buffer

当cpu需要的数据在其他cpu的cache内时，需要请求，并且等待响应，这显然是一个同步行为，优化的方案也很明显，采用异步。
思路大概是在cpu和cache之间加一个store buffer，cpu可以先将数据写到store buffer，同时给其他cpu发送消息，然后继续做其它事情，
等到收到其它cpu发过来的响应消息，再将数据从store buffer移到cache line。

### 存在问题

假设：初始状态下，假设a，b值都为0，并且a存在cpu1的cache line中(Shared状态)

```cpp
a = 1;
b = a + 1;
assert(b == 2);
```

1. cpu0 要写入a，将a=1写入store buffer，并发出Read Invalidate消息，继续其他指令。
2. cpu1 收到Read Invalidate，返回Read Response(包含a=0的cache line)和Invalidate ACK，cpu0 收到Read Response，更新cache line(a=0)。
3. cpu0 开始执行b=a+1，此时cache line中还没有加载b，于是发出Read Invalidate消息，从内存加载b=0，同时cache line中已有a=0，于是得到b=1，状态为Modified状态。
4. cpu0 得到 b=1，断言失败。
5. cpu0 将store buffer中的a=1推送到cache line，然而为时已晚。

造成这个问题的根源在于对同一个cpu存在对a的两份拷贝，一份在cache，一份在store buffer

### Store Forwarding

store buffer可能导致破坏程序顺序的问题，硬件工程师在store buffer的基础上，又实现了”store forwarding”技术: cpu可以直接从store buffer中加载数据，
即支持将cpu存入store buffer的数据传递(forwarding)给后续的加载操作，而不经由cache。

解决了同一个cpu读写数据的问题，但是还有问题，在并发场景

```cpp
void foo() {
    a = 1;
    b = 1;
}
void bar() {
    while (b == 0) continue;
    assert(a == 1)
}
```

### 新的问题

假设：假设a，b值都为0，a存在于cpu1的cache中，b存在于cpu0的cache中，均为Exclusive状态，cpu0执行foo函数，cpu1执行bar函数

1. cpu1执行while(b == 0)，由于cpu1的Cache中没有b，发出Read b消息
2. cpu0执行a=1，由于cpu0的cache中没有a，因此它将a(当前值1)写入到store buffer并发出Read Invalidate a消息
3. cpu0执行b=1，由于b已经存在在cache中，且为Exclusive状态，因此可直接执行写入
4. cpu0收到Read b消息，将cache中的b(当前值1)返回给cpu1，将b写回到内存，并将cache Line状态改为Shared
5. cpu1收到包含b的cache line，结束while (b == 0)循环
6. cpu1执行assert(a == 1)，由于此时cpu1 cache line中的a仍然为0并且有效(Exclusive)，断言失败
7. cpu1收到Read Invalidate a消息，返回包含a的cache line，并将本地包含a的cache line置为Invalid，然而已经为时已晚。
8. cpu0收到cpu1传过来的cache line，然后将store buffer中的a(当前值1)刷新到cache line

出现这个问题的原因在于cpu不知道a, b之间的数据依赖，cpu0对a的写入需要和其他cpu通信，因此有延迟，而对b的写入直接修改本地cache就行，因此b比a先在cache中生效，导致cpu1读到b=1时，a还存在于store buffer中。