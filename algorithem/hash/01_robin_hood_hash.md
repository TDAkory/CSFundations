# [Robin Hood hashing](https://github.com/martinus/robin-hood-hashing)

Robin Hood hashing 是一种用于实现散列表的开放寻址算法，它通过一种称为“偷窃”的机制来减少桶之间的探查距离差异，从而提高散列表的性能。它在处理哈希冲突时，通过移动已存在的元素来为新元素腾出空间，确保新元素能够找到一个相对靠前的位置，减少后续查找的平均探查次数。

## 核心思想

Robin Hood hashing 的核心思想是：当新元素插入时，如果目标位置已被占用，新元素会与已存在的元素进行比较，如果已存在的元素比新元素探查的次数更多（即已存在的元素的探查距离更长），新元素会“偷窃”已存在元素的位置，将其推向更远的位置。这个过程一直持续到找到一个合适的位置或者已存在元素的探查距离小于等于新元素的探查距离。

## 插入过程

以下是一个简化的插入过程描述：

1. **计算哈希值**：计算新元素的哈希值，确定其在散列表中的初始位置。
2. **探查序列**：从初始位置开始，按照一定的探查序列（如线性探查）寻找空闲位置。
3. **比较探查距离**：对于每个已存在的元素，比较其探查距离（即从初始位置到当前位置的步数）与新元素的探查距离。
4. **偷窃机制**：如果已存在的元素探查距离更长，新元素取代其位置，已存在的元素被重新插入到后续位置。
5. **重复**：重复上述过程，直到找到空闲位置或完成插入。

## 实现示例

以下是一个简化的 Robin Hood hashing 实现示例，展示了插入和查找操作：

```cpp
#include <iostream>
#include <vector>
#include <optional>

template<typename K, typename V>
class RobinHoodHashTable {
public:
    using key_type = K;
    using value_type = V;
    using bucket_type = std::pair<K, V>;

private:
    std::vector<std::optional<bucket_type>> table;
    size_t size;

    static constexpr float max_load_factor = 0.75;

public:
    RobinHoodHashTable(size_t initial_capacity = 16)
        : table(initial_capacity), size(0) {}

    size_t hash(const K& key) const {
        return std::hash<K>{}(key) % table.size();
    }

    void insert(const K& key, const V& value) {
        if (load_factor() >= max_load_factor) {
            rehash(table.size() * 2);
        }

        size_t probe_distance = 0;
        size_t index = hash(key);

        while (true) {
            if (!table[index]) {
                table[index] = {key, value};
                ++size;
                break;
            }

            const auto& existing = table[index].value();
            size_t existing_probe_distance = hash(existing.first) == index ? 0 : index - hash(existing.first);

            if (existing_probe_distance > probe_distance) {
                // Robin Hood偷窃机制
                std::swap(table[index], std::optional<bucket_type>{bucket_type(key, value)});
                std::swap(key, existing.first);
                value = existing.second;
                probe_distance = existing_probe_distance;
            }

            index = (index + 1) % table.size();
            ++probe_distance;
        }
    }

    std::optional<V> find(const K& key) const {
        size_t index = hash(key);
        size_t probe_distance = 0;

        while (table[index]) {
            if (table[index]->first == key) {
                return table[index]->second;
            }
            index = (index + 1) % table.size();
            ++probe_distance;
        }

        return std::nullopt;
    }

    float load_factor() const {
        return static_cast<float>(size) / table.size();
    }

    void rehash(size_t new_capacity) {
        RobinHoodHashTable new_table(new_capacity);

        for (const auto& bucket : table) {
            if (bucket) {
                new_table.insert(bucket->first, bucket->second);
            }
        }

        table.swap(new_table.table);
        size = new_table.size;
    }
};
```

优点
- **减少探查距离**：通过偷窃机制，将探查距离较长的元素推向后方，减少平均探查次数。
- **性能稳定**：相比其他开放寻址方法，如线性探查，Robin Hood hashing 在高负载下性能更稳定。
- **易于实现**：实现相对简单，适合在实际项目中应用。

缺点
- **额外开销**：每次插入操作可能需要移动多个元素，增加了插入的平均时间复杂度。
- **内存消耗**：为了保持性能，通常需要维持较低的负载因子，这可能导致内存浪费。

Robin Hood hashing 是一种高效的散列算法，特别适合需要快速查找和插入操作的场景。通过巧妙的偷窃机制，它能够在保持较高性能的同时减少内存访问的不均匀性。