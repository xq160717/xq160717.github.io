---
title: 跳表
date: 2019-03-28 15:14:31
categories: 算法
---

跳表是一种简单，高效的数据结构，实现起来成本最小，并且速度很快，其对标的是平衡树。它最大的优势是原理简单，容易实现，方便扩展，效率更高。在一些热门项目里用来代替平衡树，例如redis。

### 跳表的基本思想

一个简单的链表，如果我们要搜索其中的一个元素，我们需要从头开始遍历整个链表直到找到对应的元素，即时间复杂度是O(n)。同理，插入一个数并且保持链表有序，需要找到合适的插入位置，再执行插入，时间复杂度也是O(n)。跳表的引入就是为了解决这种问题，提高检索以及插入速度，下图充分反应了跳表的实现原理，看起来并没有什么难度。

<img src="skiplist.png">

当我们需要查找数字91的时候，从最上层开始从左往右遍历，到最顶层的最后一个发现67<91，指针下移。继续从左往右遍历，发现下一个元素92>91，指针继续下移。然后继续开始往右遍历找到91，如果到了最底层从左往右遍历到比自己大的元素之后还没找到，则元素不存在。查找路径如下图

<img src="lujin.png">

可以看到比起链表从左往右遍历节省了很多时间

### Java实现跳表

节点定义

```java
public class SkipListNode {

    // 节点key
    public Integer key;

    // 上方节点指针
    public SkipListNode up;

    // 下方节点指针
    public SkipListNode down;

    // 左边节点指针
    public SkipListNode left;

    // 右边节点指针
    public SkipListNode right;

    SkipListNode(Integer key) {
        this.key = key;
        up = down = left = right = null;
    }

}
```

操作

```java
/**
 * 跳表
 */
public class SkipList {

    // 用来判断是否需要增加高度的随机函数
    private Random random;

    // 跳表头节点
    private SkipListNode head;

    // 跳表尾节点
    private SkipListNode tail;

    // 跳表层数
    private int level;

    // 跳表总节点数
    private int nodes;

    // 跳表向上提升一级的概率
    private static final double PROBABILITY = 0.5;

    public SkipList() {
        random = new Random();
        head = new SkipListNode(null);
        tail = new SkipListNode(null);
        head.right = tail;
        tail.left = head;
        nodes = 0;
        level = 0;

    }

    /**
     * 插入节点
     *
     * @param key 节点值
     */
    public void insert(int key) {
        // 找到插入位置的前一个节点
        SkipListNode preNode = findFirst(key);
        // 新建一个节点
        SkipListNode node = new SkipListNode(key);
        // 插入步骤
        node.right = preNode.right;
        node.left = preNode;
        preNode.right.left = node;
        preNode.right = node;
        // 当前节点所有的层数，因为是在最下面插入的，所以初始值为0
        int nowLevel = 0;
        // 判断是否需要增加高度，一般来说每一层的节点将会是下面一层的1／2
        while (random.nextDouble() < PROBABILITY) {
            // 如果当前插入的节点所处的层数大于等于最大的层数，那么就需要增加高度，保证头尾节点的高度是最高的
            if (nowLevel >= level) {
                SkipListNode p1 = new SkipListNode(null);
                SkipListNode p2 = new SkipListNode(null);
                p1.right = p2;
                p1.down = head;
                p2.left = p1;
                p2.down = tail;
                // 将头尾节点往上移，成为最顶层的节点
                head.up = p1;
                tail.up = p2;
                // 重新赋值头尾节点
                head = p1;
                tail = p2;
                // 最高层数加一
                level++;
            }
            // 将当前节点增加高度，即在插入节点的上方插入一个新的节点，然后将其与插入的节点相连
            // 建立需要与"新插入节点上面那个节点"的链接，这里我们将新插入节点的前面同等高度的节点与之链接
            while (preNode.up == null) {
                preNode = preNode.left;
            }
            // 找到上一层中的需要链接的前一个节点
            preNode = preNode.up;
            // 新建一个上层节点
            SkipListNode e = new SkipListNode(key);
            // 插入
            e.left = preNode;
            e.right = preNode.right;
            preNode.right.left = e;
            preNode.right = e;

            e.down = node;
            node.up = e;
            // 将新插入的节点改为最上面这个
            node = e;
            // 增加高度
            nowLevel++;

        }
        // 节点加1
        nodes++;
    }


    /**
     * 在最下面那层，找到要插入的位置的前面的那个key
     *
     * @param key 需要查找的key
     * @return 节点
     */
    private SkipListNode findFirst(int key) {
        SkipListNode p = head;
        while (true) {
            // 判断要插入的位置，当没有查到尾节点并且要插入的数据还是比前面大，右移
            while (p.right.key != null && p.right.key <= key) {
                p = p.right;
            }
            // 上层没找到往下移，到达最底层跳出即可
            if (p.down != null) {
                p = p.down;
            } else {
                break;
            }
        }
        return p;
    }

    /**
     * 打印每一层节点
     */
    public void display() {
        SkipListNode node;
        SkipListNode node1 = head;
        while (node1 != null) {
            int k = 0;
            node = node1;
            while (node != null) {
                System.out.print(node.key + "\t");
                k++;
                node = node.right;
            }
            System.out.print("\t");
            System.out.print("(" + k + ")");
            System.out.println();
            node1 = node1.down;
        }
    }
}

```

Test

```java
    public static void main(String[] args) {
        SkipList skipList = new SkipList();
        Random random = new Random();
        for (int i = 0; i < 10; i++) {
            int value = (int) (random.nextDouble() * 1000);
            skipList.insert(value);
        }
        skipList.display();
    }
```

结果

<img src="result.png">

### 总结

- 跳表这种高效率的搜索方式是通过空间换时间得到的
- 跳表最终形成的数据结构和搜索树很相似
- 跳表通过抛硬币的方式来决定新插入的节点是否需要建立索引即上升一个层
- 跳表搜索的时间复杂度是O(logn)
- 没听过但是很犀利的数据结构。。







