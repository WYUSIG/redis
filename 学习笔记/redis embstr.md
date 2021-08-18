## Redis源码学习-6.0.15

### 嵌入式字符串
>嵌入式字符串是redis在创建字符串对象时，如果该字符长度少于等于44字节，则直接放到redis object *pr指针里面

#### redis object定义

```c
/* server.h
 * :4标识分配4bit(位)，又是一个节省内存的好技巧, redisObject(键值对对象) */
#define LRU_BITS 24

typedef struct redisObject {
    unsigned type:4; //OBJ_STRING
    unsigned encoding:4; //OBJ_ENCODING_EMBSTR
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;  //数据指针
} robj;
```

#### redis object type
```c
//server.h
/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
```
可以看出就是对于我们使用的5种数据结构


#### redis object encoding
```c
//server.h
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
```
对应底层的数据结构

#### 创建字符串
```c
//object.c
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44

//创建字符串对象，嵌入式字符串还是sds
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        //如果字符串长度小于等于44字节，创建嵌入式字符串，直接存在*ptr指针
        return createEmbeddedStringObject(ptr,len);
    else
        //创建普通sds字符串
        return createRawStringObject(ptr,len);
}

/* object.c 创建嵌入式字符串 */
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
    //创建redisObject，并分配空间，额外分配sds结构体长度+字符串长度+1的空间
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
    //
    struct sdshdr8 *sh = (void*)(o+1);

    //字符串类型
    o->type = OBJ_STRING;
    //编码为OBJ_ENCODING_EMBSTR，嵌入式字符串
    o->encoding = OBJ_ENCODING_EMBSTR;
    //
    o->ptr = sh+1;
    o->refcount = 1;
    //缓存淘汰策略
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }

    //sds各项属性赋值
    sh->len = len;
    sh->alloc = len;
    sh->flags = SDS_TYPE_8;
    if (ptr == SDS_NOINIT)
        //不用初始化
        sh->buf[len] = '\0';
    else if (ptr) {
        //字符串不为空，需要初始化
        memcpy(sh->buf,ptr,len);
        sh->buf[len] = '\0';
    } else {
        //填充0
        memset(sh->buf,0,len+1);
    }
    return o;
}

//走创建sds字符串逻辑
robj *createRawStringObject(const char *ptr, size_t len) {
    return createObject(OBJ_STRING, sdsnewlen(ptr,len));
}

//设置redis各项属性值
robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;

    /* Set the LRU to the current lruclock (minutes resolution), or
     * alternatively the LFU counter. */
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }
    return o;
}
```

#### 为什么是限制44字节

在 OBJ_ENCODING_EMBSTR_SIZE_LIMIT 定义我们可以看到这句注释
```c
/* The current limit of 44 is chosen so that the biggest string object
 * we allocate as EMBSTR will still fit into the 64 byte arena of jemalloc. */
```
我们再回顾一下redis object结构
```c
#define LRU_BITS 24

typedef struct redisObject {
    unsigned type:4; 
    unsigned encoding:4; 
    unsigned lru:LRU_BITS;
    int refcount;
    void *ptr; //32位系统4byte,64位系统8byte
} robj;
```
4bit + 4bit + 24bit + 4byte + 8byte= 16byte
我们再回顾一下sds8结构
```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;
    uint8_t alloc;
    unsigned char flags;
    char buf[];
};
```
头部：8bit + 8bit + 1byte = 3byte

数据部分buf[]：末尾填充'\0' = 1byte

所以在64位系统中：

64 - 16 - 3 - 1 = 44
