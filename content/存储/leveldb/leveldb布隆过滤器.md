---
写作年份:
  - "2025"
imageNameKey: leveldb布隆过滤器
tags:
  - leveldb
  - 数据库
---

Bloom Filter是一种空间效率很高的随机数据结构，它利用位数组很简洁地表示一个集合，并能判断一个元素是否属于这个集合。Bloom Filter的这种高效是有一定代价的：在判断一个元素是否属于某个集合时，有可能会把不属于这个集合的元素误认为属于这个集合（false positive）。因此，Bloom Filter不适合那些“零错误”的应用场合。而在能容忍低错误率的应用场合下，Bloom Filter通过极少的错误换取了存储空间的极大节省。

leveldb中利用布隆过滤器判断指定的key值是否存在于sstable中，若过滤器表示不存在，则该key一定不存在，由此加快了查找的效率。


# 结构

bloom过滤器底层是一个位数组，初始时每一位都是0

![布隆过滤器数组](IMG-2025-06-24-10.jpg)

当插入值x后，分别利用k个哈希函数利用x的值进行散列，并将散列得到的值与bloom过滤器的容量进行取余，将取余结果所代表的那一位值置为1。

![布隆过滤器](IMG-2025-06-24-10-1.jpg)

一次查找过程与一次插入过程类似，同样利用k个哈希函数对所需要查找的值进行散列，只有散列得到的每一个位的值均为1，才表示该值“**有可能**”真正存在；反之若有任意一位的值为0，则表示该值一定不存在。例如y1一定不存在；而y2可能存在。



![布隆过滤器哈希](IMG-2025-06-24-10-2.jpg)

布隆过滤器由两部分组成：

1. **位数组（Bit Array）**
   - 长度为 m，初始所有位为 `0`。
   - 用于存储元素的哈希信息。
2. **多个哈希函数（Hash Functions）**
   - 通常使用 kk 个独立且均匀分布的哈希函数 h1,h2,…,hkh1​,h2​,…,hk​。        
   - 每个哈希函数将输入映射到 {0,1,…,m−1}{0,1,…,m−1} 中的一个位置。

布隆过滤器通将一个key通过多个hash映射到多个数组bit，由于hash冲突的存在，映射到的bit位为1不能认为一定存在，而bit位为0则一定没有插入过，可以确定key不存在。

# 2 数学结论

首先，与布隆过滤器准确率有关的参数有：

- 哈希函数的个数k；
- 布隆过滤器位数组的容量m;
- 布隆过滤器插入的数据数量n;

主要的数学结论有：

1. 为了获得最优的准确率，当k = ln2 * (m/n)时，布隆过滤器获得最优的准确性；

2. 在哈希函数的个数取到最优时，要让错误率不超过є，m至少需要取到最小值的1.44倍；

## 2.1 误判率

布隆过滤器可能误判（返回“可能存在”但实际不存在），其误判率 pp 由以下因素决定：

- **位数组大小 m**（越大，误判率越低）
- **哈希函数数量 k**（过多或过少都会影响性能）
- **已插入元素数量 n**（元素越多，冲突概率越高）

# 3 实现

代码实现见`bloom.cc`中`class BloomFilterPolicy : public FilterPolicy`

主要有三个接口：

1. 构造器
2. `CreateFilter`:创建过滤器，根据传入的 keys，计算对应的 hash
3. `KeyMayMatch`：查找传入的 key 是否存在

## 3.1 构造器

```c++
//记录bits_per_key，并根据bits_per_key计算最有的hash个数
explicit BloomFilterPolicy(int bits_per_key) : bits_per_key_(bits_per_key) {
    // We intentionally round down to reduce probing cost a little bit
    k_ = static_cast<size_t>(bits_per_key * 0.69);  // 0.69 =~ ln(2)
    // k位于[1,30]
    if (k_ < 1) k_ = 1;
    if (k_ > 30) k_ = 30;
  }
```

## 3.2 CreateFilter

```c++
/*n是key的个数，每个key使用Slice表示，dst是保存布隆过滤器计算的结果*/
void CreateFilter(const Slice* keys, int n, std::string* dst) const override {
    // Compute bloom filter size (in both bits and bytes)
    size_t bits = n * bits_per_key_;

    // For small n, we can see a very high false positive rate.  Fix it
    // by enforcing a minimum bloom filter length.
    if (bits < 64) bits = 64;

    size_t bytes = (bits + 7) / 8; //向上取整
    bits = bytes * 8; //根据取整值重新计算bits

    const size_t init_size = dst->size();
    dst->resize(init_size + bytes, 0);  // 初始化dst
    dst->push_back(static_cast<char>(k_)); 
    char* array = &(*dst)[init_size]; // 为什么要从init_size的位置开始？
    for (int i = 0; i < n; i++) {
        // Use double-hashing to generate a sequence of hash values.
        // See analysis in [Kirsch,Mitzenmacher 2006].
        uint32_t h = BloomHash(keys[i]); //计算hash值，这里实现比较简单，只定义了一个hash函数
        //循环右移17 bits 
        const uint32_t delta = (h >> 17) | (h << 15);
        //将一个key映射到bloom过滤器多个位置上
        //对于给定的一个哈希值h，计算并设置布隆过滤器中对应的k_个位置为 1。这k_个位置是通过同一个基础哈希值h生成的，避免了多次调用哈希函数，提高了效率
        for (size_t j = 0; j < k_; j++) {
            // 计算当前哈希值对应的bit位置（在整个布隆过滤器中的位置）
            const uint32_t bitpos = h % bits;
            // 设置对应的bit为1
            // bitpos / 8 计算字节索引（每个字节8个bit）
            // bitpos % 8 计算字节内的bit偏移
            // 1 << (bitpos % 8)：生成一个只有目标 bit 为 1 的掩码
            array[bitpos / 8] |= (1 << (bitpos % 8));
            // 更新哈希值，使用之前计算的delta作为增量, 这确保了每次迭代生成不同的哈希值
            h += delta;
        }
    }
}
```

### 3.2.1 注解

#### 3.2.1.1 布隆哈希实现

```c++
static uint32_t BloomHash(const Slice& key) {
    return Hash(key.data(), key.size(), 0xbc9f1d34);
}
```

#### 3.2.1.2 循环右移

循环移位（Rotate）是一种特殊的位运算，它将二进制数的位向右或向左移动，溢出的位会被补到另一端，而不是直接丢弃（普通移位操作会丢弃溢出位）。

**数学原理**

对于一个 **n 位** 的二进制数，循环右移 **k 位** 可以分解为：

1. **右移 k 位**：得到低 k 位的部分。
2. **左移 (n-k) 位**：得到高 (n-k) 位的部分。
3. **按位或**：将两部分合并。

**公式**

```bash
rotate_right(x, k) = (x >> k) | (x << (n - k))
```

**LevelDB 选择循环右移 17 位的原因**

1. **位分布均匀性**：17 和 32（32 位整数）互质，循环右移 17 位可以确保生成的 `delta` 与原哈希值 `h` 差异足够大，从而通过加法生成的多个哈希值能均匀分布在位空间中。
2. **计算效率**：只需一次右移、一次左移和一次按位或，避免复杂计算。
3. **确定性**：同样的输入总是生成相同的输出序列，保证布隆过滤器的一致性。

# 4 参考

1. https://izualzhy.cn/leveldb-bloom-filter
1. https://www.eecs.harvard.edu/~michaelm/postscripts/tr-02-05.pdf
