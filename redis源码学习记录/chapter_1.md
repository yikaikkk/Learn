# Redis中的字符串

## Redis 字符串数据结构

在 Redis 中，字符串是一个重要的数据结构。虽然 Redis 是一个用 C 语言开发的应用，但它并没有使用 C 语言中常见的存储字符串的方式（如 `char*`），而是选择了另一种方式来处理字符串。

在 Redis 的项目结构中，文件结构采用平铺的形式。Redis 的字符串结构定义在 `/redis/redis/src` 目录下的 `sds.h` 文件中，其中包含五种字符串结构：`sdshdr5`、`sdshdr8`、`sdshdr16`、`sdshdr32` 和 `sdshdr64`。

 `sdshdr8` 的结构如下：

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;    /* 已使用的长度 */
    uint8_t alloc;  /* 分配的长度（不包括头部和空终止符） */
    unsigned char flags; /* 类型的低 3 位，未使用的高 5 位 */
    char buf[];     /* 字符串内容 */
};
```

### 为什么redis不使用char* 
对比c语言提供的char*的方式来存储字符串，redis使用的数据结构，除了保存sds的数据类型flag和字符串内容以外，还保存了现在使用的数据长度，已经分配的长度。

在c语言中，以char*保存字符串，在保存字符串时是一个来续的内存空间，以`\0`结尾。
在计算长度时，在使用char*的方式中，计算长度时，会依次循环检查每个字符是否是`\0`来判断是否是结尾。
对于在redis中的字符串来说，结尾并不是一定以`\0`结尾，因此如果使用char*来保存字符串，在获取长度是可能会被截断。

同时
redis中的长度计算函数如下
```c
static inline size_t sdslen(const sds s) {
    switch (sdsType(s)) {
        case SDS_TYPE_5: return SDS_TYPE_5_LEN(s);
        case SDS_TYPE_8:
            return SDS_HDR(8,s)->len;
        case SDS_TYPE_16:
            return SDS_HDR(16,s)->len;
        case SDS_TYPE_32:
            return SDS_HDR(32,s)->len;
        case SDS_TYPE_64:
            return SDS_HDR(64,s)->len;
    }
    return 0;
}
```
在获取字符串长度时，以char*方式保存的数据获取时间复杂度为O(n)，但是在sds的方式下，获取的时间复杂度为O(1)，。

对于别的函数，例如追加
```c
sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s);

    s = sdsMakeRoomFor(s,len);
    if (s == NULL) return NULL;
    memcpy(s+curlen, t, len);
    sdssetlen(s, curlen+len);
    s[curlen+len] = '\0';
    return s;
}
```
在redis中是直接在后面动态分配空间，在复制进去，不需要像c语言的实现先遍历再添加

### 总结
sds相对于char*的优势：

1、对于元数据的保存，在需要获取长度时，以O(1)的时间获取，效率提升

2、以长度来表示结尾，避免因为`\0`导致字符串被截断




