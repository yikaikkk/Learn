# Redis中的数据结构
## 有序集合
在redis中，还有一个特殊的数据结构，就是有序集合，有序集合是一个很重要的数据结构

和之前的数据结构不同，zset的定义在server.h

```c
typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```

由数据结构的定义来看，zset是由dict和zskiplist两种数据结构组合而成。因此通过数据结构的自身的性质来说，dict提供了O(1)的单点查询效率和O(logN)+M的范围查询效率

### skiplist

跳表是一种多重链表的数据数据结构

![跳表](./static/skiplist.jpg "Magic Gardens")

在redis中的跳表的定义如下

```c
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;
```

在跳表中保存了两个数据字段，一个是sds类型的ele，就是元素本身，另一个是double类型的数据权重score，同时跳表是一个双向链表，使用backward来保存了前一个节点，只记录了level0上的前一个节点，zskiplistLevel是一个结构体，对应了跳表的每一层，其中span对应了跨越了level0上个数

## 跳表查询

对于跳表的查询实际上就是对多层链表的查询