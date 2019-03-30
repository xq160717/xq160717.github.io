---
title: 布隆过滤器
date: 2019-03-28 15:14:50
categories: 算法
---

### 什么是布隆过滤器

布隆过滤器是一种比较巧妙的概率型数据结构，特点是高效的插入和和查询，可以用来告诉我们“某样东西一定不存在或者可能存在”。相比较于传统的数据结构，map，list，set，他更高效，但缺点是其返回结果是概率性的而不是确切的。

基于这种“某样东西一定不存在或者存在”的特性，我们可以很方便的做到在大数据量的情况下进行去重操作。

### 布隆过滤器原理

原理其实也是非常的简单，其核心是一个超大的位数组和几个哈希函数，当存入一个元素的时候，使用每一个哈希函数进行hash，再与位数组取模，得出的位置置为1。当判断一个元素是否存在的时候也是用这种方法判断对应的数组位是不是为1。至此我们就可以得出下面两个结论了

- 不存在则一定不存在：这个特性用来去重非常方便
- 存在不一定真的存在：可能发生了key的碰撞

下面是一个具体的例子，首先是布隆过滤器的基本数据结构位数组，长这个样子

<img src="jiben.png">

当我们要映射一个值到布隆过滤器中的时候，我们需要使用多个不同的hash函数生成多个hash值，并对每个生成的hash值指向的bit位置为1，例如针对值“abc”和三个不同的hash函数分别生产了hash值1，5，6，则上图转变为

<img src="1.png">

我们再存一个值“mnb”，如果hash函数返回1，4，7的话，图继续变为

<img src="2.png">

我们可以看到，1这个bit位，由于两个值的hash函数都返回了这个位，因此他被覆盖了。现在如果我们想查询“xyz”这个值是否存在的话，假设hash函数返回了1，5，6三个值，结果我们发现5这个bit位的值为0，说明没有任何一个值映射到这个bit位上，因此我们可以很确定的说“xyz”这个值不存在。再继续假设，我们需要查询“abc”是否存在的话，那么hash函数返回1，5，6然后我们发现这三个bit位上的值均为1，但是，我们依旧不可以说“abc”这个值存在，只能说“abc”这个值可能存在，因为可能产生了hash碰撞。

### 超简易版本实现

下面给出自己对布隆过滤器的超级简易的实现

```java
public class BloomFilter {

    // 位数组大小
    private static final int DEFAULT_SIZE = 2 << 24;

    // 简单的根据几个不同的种子来构建不同的hash函数
    private static final int[] seeds = new int[] {7, 11, 13, 31, 37, 61,};

    // 位数组
    private BitSet bits = new BitSet(DEFAULT_SIZE);

    // hash函数
    private SimpleHash[] func = new SimpleHash[seeds.length];

    public static void main(String[] args) {
        String value = "923797211@qq.com";
        BloomFilter filter = new BloomFilter();
        System.out.println(filter.contains(value));
        filter.add(value);
        System.out.println(filter.contains(value));
    }

    public BloomFilter() {
        for (int i = 0; i < seeds.length; i++) {
            func[i] = new SimpleHash(DEFAULT_SIZE, seeds[i]);
        }
    }


    public void add(String value) {
        for (SimpleHash f : func) {
            bits.set(f.hash(value), true);
        }
    }

    public boolean contains(String value) {
        if (value == null) {
            return false;
        }
        boolean ret = true;
        for (SimpleHash f : func) {
            ret = ret && bits.get(f.hash(value));
        }
        return ret;
    }


    public static class SimpleHash {

        private int cap;

        private int seed;

        public SimpleHash(int cap, int seed) {
            this.cap = cap;
            this.seed = seed;
        }

        public int hash(String value) {
            int result = 0;
            int len = value.length();
            for (int i = 0; i < len; i++) {
                result = seed * result + value.charAt(i);
            }
            return (cap - 1) & result;
        }

    }

}
```

