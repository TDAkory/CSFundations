# Cache及优化手段

## Read List

- [如何用 Cache 在 C++ 程序中进行系统级优化（一）](https://toutiao.io/posts/60r22r/preview)
- [如何用 Cache 在 C++ 程序中进行系统级优化（二）](https://toutiao.io/posts/1e33zc/preview)

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
