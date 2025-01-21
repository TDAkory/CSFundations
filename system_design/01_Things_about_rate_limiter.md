# Rate Limiter

## 限流算法

|算法|描述|关键参数|优点|缺点|
|-------------|-------------|-------------|-------------|-------------|
|令牌桶 Token Bucket|按照一定速率向桶中添加令牌，桶装满则多余的令牌丢弃，请求需消耗令牌。|桶大小和填充速率|可以应对一定程度的流量突增|在流量变化时，如何权衡桶大小和填充率|
|漏桶 Leaking Bucket|请求填充到桶里，桶满了则拒绝请求。桶里的元素以固定的速率流出|桶大小和消费速率|使用丢列易实现|消费速率固定，存在效率问题，突发的流量会用旧的请求填满队列,如果不及时处理，最近的请求将受到速率限制 |
|固定窗口计数 Fixed Window Counter| 时间维度划分窗口，每个窗口允许通过固定数量的请求，超出则会拒绝。||简单易于理解|时间窗口边缘应对流量高峰时，可能会让通过的请求数超过设置的阈值|
|滑动窗口日志 Sliding Window Log|每个请求记录请求时间，过期删除日志。请求通过时需要判断集合内的日志数不超过阈值||限速准确|额外内存开销，即使一个请求被拒绝,它的时间戳仍可能存储在内存中 |
|活动窗口计数 Sliding Window Counter|在固定窗口的基础上拆分更小的子窗口，每次滑动一个子窗口的范围||避免超出限制||

### 令牌桶Demo

```cpp
#include <iostream>
#include <chrono>
#include <thread>
#include <mutex>
#include <condition_variable>

class TokenBucket {
public:
    TokenBucket(size_t bucket_size, double rate)
        : bucket_size_(bucket_size), rate_(rate), tokens_(bucket_size), last_update_time_(std::chrono::system_clock::now()) {}

    bool try_acquire_tokens(size_t requested_tokens) {
        std::unique_lock<std::mutex> lock(mutex_);
        refill_tokens();
        if (tokens_ >= requested_tokens) {
            tokens_ -= requested_tokens;
            return true;
        }
        return false;
    }

private:
    size_t bucket_size_;
    double rate_;
    size_t tokens_;
    std::chrono::system_clock::time_point last_update_time_;
    std::mutex mutex_;
    std::condition_variable cv_;

    void refill_tokens() {
        auto now = std::chrono::system_clock::now();
        auto elapsed = std::chrono::duration_cast<std::chrono::seconds>(now - last_update_time_).count();
        double new_tokens = elapsed * rate_;
        tokens_ = std::min(tokens_ + static_cast<size_t>(new_tokens), bucket_size_);
        last_update_time_ = now;
    }
};
```

### 漏桶Demo

```cpp
#include <iostream>
#include <chrono>
#include <thread>

// 类名首字母大写，使用驼峰命名法
class LeakyBucket {
 public:
  // 构造函数，使用冒号初始化列表，明确参数名称
  LeakyBucket(double capacity, double rate) : capacity_(capacity), rate_(rate), water_(0.0) {
    last_leak_time_ = std::chrono::steady_clock::now();
  }

  // 尝试向桶中添加水量，函数名使用小写加下划线的风格
  bool TryAddWater(double amount) {
    // 先漏水
    double leaked = CalculateLeak();
    water_ = std::max(0.0, water_ - leaked);
    UpdateLeakTime();

    if (water_ + amount <= capacity_) {
      water_ += amount;
      return true;
    } else {
      return false;
    }
  }

 private:
  // 成员变量以下划线结尾，私有成员变量使用下划线命名法
  double capacity_;
  double rate_;
  double water_;
  std::chrono::steady_clock::time_point last_leak_time_;

  // 计算从上次漏水到现在应该漏出的水量
  double CalculateLeak() {
    auto now = std::chrono::steady_clock::now();
    std::chrono::duration<double> duration = now - last_leak_time_;
    double leaked = rate_ * duration.count();
    return leaked;
  }

  // 更新漏水时间
  void UpdateLeakTime() {
    last_leak_time_ = std::chrono::steady_clock::now();
  }
};
```

### 固定窗口计数Demo

```cpp
#include <iostream>
#include <chrono>
#include <thread>

class FixedWindowRateLimiter {
 public:
  // 构造函数，使用冒号初始化列表，明确参数名称
  FixedWindowRateLimiter(int max_requests, int window_seconds)
      : limit_(max_requests),
        window_size_(std::chrono::seconds(window_seconds)),
        count_(0),
        window_start_(std::chrono::system_clock::now()) {}

  // 尝试请求，函数名使用小写加下划线的风格
  bool TryRequest() {
    CheckAndReset();
    if (count_ < limit_) {
      ++count_;
      return true;
    } else {
      return false;
    }
  }

 private:
  // 成员变量以下划线结尾，私有成员变量使用下划线命名法
  int limit_;
  std::chrono::seconds window_size_;
  int count_;
  std::chrono::time_point<std::chrono::system_clock> window_start_;

  // 检查是否需要重置窗口
  void CheckAndReset() {
    auto now = std::chrono::system_clock::now();
    if (now - window_start_ >= window_size_) {
      count_ = 0;
      window_start_ = now;
    }
  }
};
```

### 活动窗口日志Demo

```cpp
#include <iostream>
#include <deque>
#include <chrono>
#include <thread>

class SlidingWindowRateLimiter {
public:
    // 构造函数，初始化限流的最大请求数和窗口大小（秒）
    SlidingWindowRateLimiter(int max_requests, int window_size_seconds) :
        max_requests_(max_requests), window_size_(std::chrono::seconds(window_size_seconds)) {}

    // 尝试进行一次请求，返回是否允许该请求
    bool try_request() {
        auto now = std::chrono::system_clock::now();
        // 移除窗口外的旧请求记录
        remove_old_requests(now);
        // 检查当前请求数量是否超过限制
        if (requests_.size() < max_requests_) {
            requests_.push_back(now);
            return true;
        }
        return false;
    }

private:
    int max_requests_;
    std::chrono::seconds window_size_;
    std::deque<std::chrono::time_point<std::chrono::system_clock>> requests_;

    // 移除窗口外的旧请求记录
    void remove_old_requests(const std::chrono::time_point<std::chrono::system_clock>& now) {
        while (!requests_.empty() && (now - requests_.front() >= window_size_)) {
            requests_.pop_front();
        }
    }
};
```

## 分布式场景

上述算法在单节点下很好实现，其中令牌桶和滑动窗口是两种较好的选择，能够一定程度上处理不均匀的请求分布。但在分布式场景下，仅考虑算法是不够的。

算法配置位置：

1. 配置在负载均衡器
   1. 这个方式的限流是全局性的。但是涉及到对负载均衡器进行修改，可能破坏“职责单一”，缺少”容错能力“，功能耦合导致”不方便扩容“

2. 配置在中心化数据库
   1. 服务流量不大的情况下，这种简单的方式是可行的
   2. 服务流量上涨之后，每个请求都需要请求数据库并等待数据库的响应，导致数据库压力增大，处理延迟增加

3. 优化基于中心化数据库的限流器
   1. 分布式限流：减少访问中心的频率，在分布式节点上做一部分限流器的工作，再周期性地访问数据库同步。因此将限流器分为本地限流器 + 中心限流器
   2. Option-1：在节点上积攒一定数量的请求N再去请求中心限流器。优点是可以降低对中心化数据库的访问频率；缺点是积攒过程会增加请求的处理延迟
   3. Option-2：利用技术手段将流量相对均衡的分布在各个节点上，因此`本地限流器的配置 = 中心限流器的配置 / 节点数量`。本地限流器与中心限流器保持心跳以便更新配置，中心限流器根据心跳统计活跃节点数进行额度计算。缺点是可能存在误差，不够准确
   4. Option-3：本地限流器初始化时向中心限流器请求N个令牌，作为自己的令牌，消耗完之后再去请求。优点是不会将请求阻塞在积攒过程上；缺点是如果N设置不合理，则某些节点可能会饿死

## 工程评估

限流器在哪里实现？在服务器端还是在网关中？没有绝对的答案。这取决于目标服务当前的技术堆栈、工程资源、优先级、目标等:

* 首先评估技术可行性，程序语言、缓存服务等，确保能够有效的实现服务端限流
* 识别适合业务需求的速率限制算法。在服务器端实现一切时，就可以完全控制算法了。但是，如果使用第三方网关，选择可能会受到限制。
* 如果已经使用微服务架构，并在设计中包含API网关来执行身份验证、IP白名单等，可以在API网关中添加速率限制器。
* 建立自己的限速服务需要时间。如果没有足够的工程资源来实现一个速率限制器，一个商业的API网关是一个更好的选择。

对于限流被拒绝时的处理方式，取决于业务场景：

* 可以选择直接拒绝请求：比如HTTP请求，返回429
* 暂缓接收：比如TCP长连接，如果关闭连接可能造成反复重连，可以利用TCP的拥塞控制，停止接收，客户端会暂缓发送
* 在限流器前安置一个队列，被拒绝的请求入队，延时一段时间再次判断

## 扩展话题

## 3rd party libraries

- [Hystrix: Latency and Fault Tolerance for Distributed Systems](https://github.com/Netflix/Hystrix)
- [Fault tolerance library designed for functional programming](https://github.com/resilience4j/resilience4j)
- []()

## Ref

- [设计一个限流组件](https://www.cnblogs.com/myshowtime/p/16313623.html)