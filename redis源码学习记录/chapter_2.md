# Redis中的数据结构

## Redis中的哈希表

在redis中哈希的头文件和实现分别是src文件夹下的dict.h和dict.c，dict的具体定义如下

```c
struct dict {
    dictType *type;

    dictEntry **ht_table[2];
    unsigned long ht_used[2];

    long rehashidx; /* rehashing not in progress if rehashidx == -1 */

    /* Keep small vars at end for optimal (minimal) struct padding */
    unsigned pauserehash : 15; /* If >0 rehashing is paused */

    unsigned useStoredKeyApi : 1; /* See comment of storedHashFunction above */
    signed char ht_size_exp[2]; /* exponent of size. (size = 1<<exp) */
    int16_t pauseAutoResize;  /* If >0 automatic resizing is disallowed (<0 indicates coding error) */
    void *metadata[];
};
```
在dict结构中，哈希表被保存为一个二维数组ht_table，dictEntry的定义如下

```c
struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;     /* Next entry in the same hash bucket. */
};
```
在redis中dictEntry就是哈希表的每个节点，有三个部分，键、值和指向下一个节点的指针next。根据代码实现可以得到，在实现哈希表时，redis处理哈希冲突的方法是采用链式哈希，当有别的值被映射到一个hash值时就采用链表的方式连接。

## 链式hash的局限和解决

采用链式处理哈希冲突时，如果数量过大，导致链表过长会导致查询效率过低，所以redis针对这种问题采用渐进式hash的方式来处理。
在dict的数据结构中，ht_table定义了两个数组，同时还保存了一个哈希的下标

```c
 dictEntry **ht_table[2];
 long rehashidx;
 ```

### 查看插入函数

```c
dictEntryLink dictFindLinkForInsert(dict *d, const void *key, dictEntry **existing) {
    unsigned long idx, table;
    dictCmpCache cmpCache = {0};
    dictEntry *he;
    uint64_t hash = dictHashKey(d, key, d->useStoredKeyApi);
    if (existing) *existing = NULL;
    idx = hash & DICTHT_SIZE_MASK(d->ht_size_exp[0]);

    /* Rehash the hash table if needed */
    _dictRehashStepIfNeeded(d,idx);

    /* Expand the hash table if needed */
    _dictExpandIfNeeded(d);
    keyCmpFunc cmpFunc = dictGetCmpFunc(d);

    for (table = 0; table <= 1; table++) {
        if (table == 0 && (long)idx < d->rehashidx) continue; 
        idx = hash & DICTHT_SIZE_MASK(d->ht_size_exp[table]);
        /* Search if this slot does not already contain the given key */
        he = d->ht_table[table][idx];
        while(he) {
            void *he_key = dictGetKey(he);
            if (key == he_key || cmpFunc(&cmpCache, key, he_key)) {
                if (existing) *existing = he;
                return NULL;
            }
            he = dictGetNext(he);
        }
        if (!dictIsRehashing(d)) break;
    }

    /* If we are in the process of rehashing the hash table, the bucket is
     * always returned in the context of the second (new) hash table. */
    dictEntry **bucket = &d->ht_table[dictIsRehashing(d) ? 1 : 0][idx];
    return bucket;
}
```
再每次插入的的时候都会检查是否需要进行rehash或者扩容。同时在插入时会有两种逻辑

```c
 if (table == 0 && (long)idx < d->rehashidx) continue; 
 ```
 在第一个ht_table，并且如果是小于rehashidx的下标就直接跳过。最后再进行检查后返回桶


### 查看rehash函数

```c

/* Performs N steps of incremental rehashing. Returns 1 if there are still
 * keys to move from the old to the new hash table, otherwise 0 is returned.
 *
 * Note that a rehashing step consists in moving a bucket (that may have more
 * than one key as we use chaining) from the old to the new hash table, however
 * since part of the hash table may be composed of empty spaces, it is not
 * guaranteed that this function will rehash even a single bucket, since it
 * will visit at max N*10 empty buckets in total, otherwise the amount of
 * work it does would be unbound and the function may block for a long time. */
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    unsigned long s0 = DICTHT_SIZE(d->ht_size_exp[0]);
    unsigned long s1 = DICTHT_SIZE(d->ht_size_exp[1]);
    if (dict_can_resize == DICT_RESIZE_FORBID || !dictIsRehashing(d)) return 0;
    /* If dict_can_resize is DICT_RESIZE_AVOID, we want to avoid rehashing. 
     * - If expanding, the threshold is dict_force_resize_ratio which is 4.
     * - If shrinking, the threshold is 1 / (HASHTABLE_MIN_FILL * dict_force_resize_ratio) which is 1/32. */
    if (dict_can_resize == DICT_RESIZE_AVOID && 
        ((s1 > s0 && s1 < dict_force_resize_ratio * s0) ||
         (s1 < s0 && s0 < HASHTABLE_MIN_FILL * dict_force_resize_ratio * s1)))
    {
        return 0;
    }

    while(n-- && d->ht_used[0] != 0) {
        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(DICTHT_SIZE(d->ht_size_exp[0]) > (unsigned long)d->rehashidx);
        while(d->ht_table[0][d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        /* Move all the keys in this bucket from the old to the new hash HT */
        rehashEntriesInBucketAtIndex(d, d->rehashidx);
        d->rehashidx++;
    }

    return !dictCheckRehashingCompleted(d);
}

/* Helper function for `dictRehash` and `dictBucketRehash` which rehashes all the keys
 * in a bucket at index `idx` from the old to the new hash HT. */
static void rehashEntriesInBucketAtIndex(dict *d, uint64_t idx) {
    dictEntry *de = d->ht_table[0][idx];
    uint64_t h;
    dictEntry *nextde;
    while (de) {
        nextde = dictGetNext(de);
        void *key = dictGetKey(de);
        /* Get the index in the new hash table */
        if (d->ht_size_exp[1] > d->ht_size_exp[0]) {
            h = dictHashKey(d, key, 1) & DICTHT_SIZE_MASK(d->ht_size_exp[1]);
        } else {
            /* We're shrinking the table. The tables sizes are powers of
             * two, so we simply mask the bucket index in the larger table
             * to get the bucket index in the smaller table. */
            h = idx & DICTHT_SIZE_MASK(d->ht_size_exp[1]);
        }
        if (d->type->no_value) {
            if (!d->ht_table[1][h]) {
                /* The destination bucket is empty, allowing the key to be stored 
                 * directly without allocating a dictEntry. If an old entry was 
                 * previously allocated, free its memory. */                
                if (!entryIsKey(de)) zfree(decodeMaskedPtr(de));
                
                if (d->type->keys_are_odd)
                    de = key; /* ENTRY_PTR_IS_ODD_KEY trivially set by the odd key. */
                else
                    de = encodeMaskedPtr(key, ENTRY_PTR_IS_EVEN_KEY);
                
            } else if (entryIsKey(de)) {
                /* We don't have an allocated entry but we need one. */
                de = createEntryNoValue(key, d->ht_table[1][h]);
            } else {
                /* Just move the existing entry to the destination table and
                 * update the 'next' field. */
                assert(entryIsNoValue(de));
                dictSetNext(de, d->ht_table[1][h]);
            }
        } else {
            dictSetNext(de, d->ht_table[1][h]);
        }
        d->ht_table[1][h] = de;
        d->ht_used[0]--;
        d->ht_used[1]++;
        de = nextde;
    }
    d->ht_table[0][idx] = NULL;
}

static int dictCheckRehashingCompleted(dict *d) {
    if (d->ht_used[0] != 0) return 0;
    
    if (d->type->rehashingCompleted) d->type->rehashingCompleted(d);
    if (d->type->bucketChanged)
        d->type->bucketChanged(d, -(long long)DICTHT_SIZE(d->ht_size_exp[0]));
    zfree(d->ht_table[0]);
    /* Copy the new ht onto the old one */
    d->ht_table[0] = d->ht_table[1];
    d->ht_used[0] = d->ht_used[1];
    d->ht_size_exp[0] = d->ht_size_exp[1];
    _dictReset(d, 1);
    d->rehashidx = -1;
    return 1;
}
```
在每次渐进式hash开始时

1、都会先根据要迁移的数量，即入参n进行n次循环，对n个桶进行迁移（调用n次rehashEntriesInBucketAtIndex函数）
2、检查原来的bucket是否为空，为空就释放d->ht_table[0]，同时把d->ht_table[1]复制到d->ht_table[0]，不为空返回1（调用dictCheckRehashingCompleted）


## 小结
1、在redis中，hash表在处理哈希冲突时采用的是链式哈希
2、当链式hash的某个桶的长度过长时，导致查询效率低下，因此redis采用渐进式hash的方式扩容
3、redis中扩容大小为现在的2倍
4、扩容条件 条件一：ht[0]的大小为 0。条件二：ht[0]承载的元素个数已经超过了 ht[0]的大小，同时 Hash 表可以进行扩容。条件三：ht[0]承载的元素个数，是 ht[0]的大小的 dict_force_resize_ratio 倍，其中，dict_force_resize_ratio 的默认值是 5。

