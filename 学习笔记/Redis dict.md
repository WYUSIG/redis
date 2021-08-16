## Redis源码学习



>Redis实现的哈希表解决哈希冲突的方式是链式哈希，以及设计了渐进式rehash，由两个内部的哈希表完成



##### 源码文件：

>dict.h、dict.c



##### dict结构体设计

```c
//哈希节点链表
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;

//哈希表一些自定义函数，如哈希函数、key判等...
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;

//index(hashcode计算得来)->dictEntry
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

//哈希表
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; //rehash进度位置，-1代表没有在进行rehash
    unsigned long iterators; //该哈希表当前运行的迭代器
} dict;
```





##### 创建哈希表

```c
/* 创建一个新的哈希表 */
dict *dictCreate(dictType *type,
        void *privDataPtr)
{
    //创建一个dict，并分配空间
    dict *d = zmalloc(sizeof(*d));

    //初始化
    _dictInit(d,type,privDataPtr);
    return d;
}

/* Initialize the hash table */
int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{
    //table=NULL、size=0、sizemask=0、used=0
    //初始化第一个哈希表的元数据值
    _dictReset(&d->ht[0]);
    //初始化第一个哈希表的元数据值
    _dictReset(&d->ht[1]);
    //类型
    d->type = type;
    //私有数据
    d->privdata = privDataPtr;
    //没有在rehash
    d->rehashidx = -1;
    //当前运行迭代器数量0
    d->iterators = 0;
    //初始化成功
    return DICT_OK;
}
```



##### 添加元素

```c
/* Add an element to the target hash table */
int dictAdd(dict *d, void *key, void *val)
{
    //创建节点
    dictEntry *entry = dictAddRaw(d,key,NULL);

    //创建节点失败
    if (!entry) return DICT_ERR;
    //设置value，可能使用valDup
    dictSetVal(d, entry, val);
    //添加成功
    return DICT_OK;
}

/**
 * 添加哈希节点
 */
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;

    //如果哈希表d正在rehash中，添加元素会进行一部分rehash工作(渐进式rehash)
    if (dictIsRehashing(d)) _dictRehashStep(d);

    //获取改key的哈希值所在table的下标，如果返回-1表示改key已经存在，不进行创建
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    //如果哈希表d正在rehash，使用第二个dictht，否则使用第一个dictht
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    //创建哈希节点并分配空间
    entry = zmalloc(sizeof(*entry));
    //指向该位置的第一个元素
    entry->next = ht->table[index];
    //头插法
    ht->table[index] = entry;
    //可使用的节点数+1
    ht->used++;

    //设置节点的key，因为可能使用keyDup函数
    dictSetKey(d, entry, key);
    //返回节点
    return entry;
}

/**
 * 如果哈希表d的type的keyDup返回不为空，则节点的key为keyDup返回值
 */
#define dictSetKey(d, entry, _key_) do { \
    if ((d)->type->keyDup) \
        (entry)->key = (d)->type->keyDup((d)->privdata, _key_); \
    else \
        (entry)->key = (_key_); \
} while(0)

/**
 * 如果哈希表d的type的valDup返回不为空，则节点的value为valDup返回值
 */
#define dictSetVal(d, entry, _val_) do { \
    if ((d)->type->valDup) \
        (entry)->v.val = (d)->type->valDup((d)->privdata, _val_); \
    else \
        (entry)->v.val = (_val_); \
} while(0)
```



##### 根据key获取节点

```c
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    uint64_t h, idx, table;

    //如果哈希表d为空，则返回NULL
    if (dictSize(d) == 0) return NULL;
    //如果哈希表d正在rehash, 则进行一部分rehash工作
    if (dictIsRehashing(d)) _dictRehashStep(d);
    //获取该哈希表下该key的hashcode
    h = dictHashKey(d, key);
    //rehash过程中的话，哈希表里面的两个table都会尝试去获取
    for (table = 0; table <= 1; table++) {
        //计算下标index
        idx = h & d->ht[table].sizemask;
        //获取该hashcode的链表
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key))
                //找到则直接返回
                return he;
            //遍历链表
            he = he->next;
        }
        //如果没有在rehash，则不用找table[1]
        if (!dictIsRehashing(d)) return NULL;
    }
    //没有找到
    return NULL;
}
```



##### 渐进式rehash

```c
/**
 * 如果哈希表d没有在迭代器遍历，才会进行渐进式rehash
 */
static void _dictRehashStep(dict *d) {
    if (d->iterators == 0) dictRehash(d,1);
}


int dictRehash(dict *d, int n) {
    //每次rehash最多搬10个节点
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    //如果哈希表没有在rehash，直接返回0
    if (!dictIsRehashing(d)) return 0;

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        //确保rehashidx不溢出，rehashidx > -1代表已经迁移到那个位置
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        //如果该位置没有元素，则加到有元素的下一个位置
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            //如果已经搬了10次，退出
            if (--empty_visits == 0) return 1;
        }
        //获取该位置的链表节点
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;

            nextde = de->next;
            //新的hashcode
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            //熟悉的头插法
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            //第一个table可使用的节点数-1
            d->ht[0].used--;
            //第二个table可使用的节点数+1
            d->ht[1].used++;
            //下一节点
            de = nextde;
        }
        //旧哈希表该位置赋值为NULL
        d->ht[0].table[d->rehashidx] = NULL;
        //搬迁位置+1
        d->rehashidx++;
    }

    //如果第一个哈希表的可用链表节点数已经为0，则把第一和第二个哈希表互换，rehashidx设置为-1，代表结束rehash
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```

##### 扩容

```c
static int _dictExpandIfNeeded(dict *d)
{
    //如果已经在rehash，则返回成功
    if (dictIsRehashing(d)) return DICT_OK;

    //如果哈希表是空的，则进行第一次扩容初始化，初始容量为4
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    //如果节点数量大于或等于哈希表容量 而且（允许扩容或者总节点数大于5倍哈希表容量），redis的哈希冲突放得比较宽
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        //扩容
        return dictExpand(d, d->ht[0].used*2);
    }
    //返回成功
    return DICT_OK;
}

/**
 * 扩容
 */
int dictExpand(dict *d, unsigned long size)
{
    //如果已经在rehash或者第一个哈希表可用节点数>扩容后容量，返回错误
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    dictht n;
    //拿到满足大于等于size的最小的2的次幂的长度，为了使用&位运算计算下标
    unsigned long realsize = _dictNextPower(size);

    //如果新容量和旧容量一样，返回错误
    if (realsize == d->ht[0].size) return DICT_ERR;

    //2的次幂
    n.size = realsize;
    //2的次幂-1
    n.sizemask = realsize-1;
    //新内部哈希表
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    //节点数0
    n.used = 0;

    //如果第一次扩容，直接放到第一个位置哈希表
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    //否则放到第二个哈希表，rehashidx赋值为0，代表正在rehash, rehash到下标0位置
    d->ht[1] = n;
    d->rehashidx = 0;
    //返回成功
    return DICT_OK;
}
```
