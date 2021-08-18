## Redis源码学习-6.0.15

### 压缩列表

压缩列表是redis为了压缩列表空白空间，定义的一种元素数据长度变长的数据结构

其列表布局为：

```c
<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>
<列表字节数> <列表最后一个元素偏移量> <列表元素个数> <元素1> ... <元素n> <末尾结束字符255>
```

其中，每个元素的布局是：

```c
<prevlen> <encoding> <entry-data>
```


#### ziplist entry结构体

```c
typedef struct zlentry {
    unsigned int prevrawlensize; /* 前一个元素的长度字节数 */
    unsigned int prevrawlen;     /* 前一个元素长度 */
    unsigned int lensize;        /* 当前元素长度字节数 */
    unsigned int len;            /* 字符串长度、数字范围 */
    unsigned int headersize;     /* prevrawlensize + lensize. */
    unsigned char encoding;      /* ZIP_STR_* or ZIP_INT_* */
    unsigned char *p;            /* Pointer to the very start of the entry, that
                                    is, this points to prev-entry-len field. */
} zlentry;
```

#### 创建空ziplist

```c
//ziplist.c
unsigned char *ziplistNew(void) {
    //计算zlbytes长度 + zltail长度 + zllen长度 + zlend长度
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;
    //分配zlbytes、zltail、zllen、zlend空间
    unsigned char *zl = zmalloc(bytes);
    //设置zlbytes
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    //设置zltail
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    //zllen初始化为0
    ZIPLIST_LENGTH(zl) = 0;
    //结尾加上结束符
    zl[bytes-1] = ZIP_END;
    //返回
    return zl;
}
```

#### 根据元素指针找出下一个元素

```c
#define ZIP_END 255

/**
 * @parm zl 压缩列表
 * @parm p 当前元素
 */
unsigned char *ziplistNext(unsigned char *zl, unsigned char *p) {
    ((void) zl);

    //如果已到压缩列表结尾字符，直接返回NULL
    if (p[0] == ZIP_END) {
        return NULL;
    }

    //指针右移当前元素长度
    p += zipRawEntryLength(p);
    //如果下一个元素到了压缩列表结尾，返回NULL
    if (p[0] == ZIP_END) {
        return NULL;
    }

    return p;
}

//返回p指向元素的byte数
unsigned int zipRawEntryLength(unsigned char *p) {
    unsigned int prevlensize, encoding, lensize, len;
    //判断prevrawlensize字节数
    ZIP_DECODE_PREVLENSIZE(p, prevlensize);
    //解析出encoding、lensize、len
    ZIP_DECODE_LENGTH(p + prevlensize, encoding, lensize, len);
    //指针右移headersize = prevrawlensize + lensize
    return prevlensize + lensize + len;
}

#define ZIP_BIG_PREVLEN 254
#define ZIP_DECODE_PREVLENSIZE(ptr, prevlensize) do {                          \
    if ((ptr)[0] < ZIP_BIG_PREVLEN) {                                          \
        (prevlensize) = 1;                                                     \
    } else {                                                                   \
        (prevlensize) = 5;                                                     \
    }                                                                          \
} while(0)
```


#### push操作

```c
/**
 * @parm *zl 压缩列表
 * @parm *s 插入的字符串
 * @parm slen 插入字符串的长度
 * @parm where 0插入头部，其他插入尾部
 */
unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where) {
    unsigned char *p;
    //如果where==0，获取ziplist entrys头部指针，否则获取最尾部指针
    p = (where == ZIPLIST_HEAD) ? ZIPLIST_ENTRY_HEAD(zl) : ZIPLIST_ENTRY_END(zl);
    //插入
    return __ziplistInsert(zl,p,s,slen);
}

/* Insert item at "p". */
unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    //zlbytes
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen;
    unsigned int prevlensize, prevlen = 0;
    size_t offset;
    int nextdiff = 0;
    unsigned char encoding = 0;
    long long value = 123456789; /* initialized to avoid warning. Using a value
                                    that is easy to see if for some reason
                                    we use it uninitialized. */
    zlentry tail;

    /* Find out prevlen for the entry that is inserted. */
    if (p[0] != ZIP_END) {
        //如果p不是压缩列表尾部的话，解析出p指向当前元素的prevlensize、prevlen
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
    } else {
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        if (ptail[0] != ZIP_END) {
            prevlen = zipRawEntryLength(ptail);
        }
    }

    /* See if the entry can be encoded */
    if (zipTryEncoding(s,slen,&value,&encoding)) {
        /* 'encoding' is set to the appropriate integer encoding */
        reqlen = zipIntSize(encoding);
    } else {
        /* 'encoding' is untouched, however zipStoreEntryEncoding will use the
         * string length to figure out how to encode it. */
        reqlen = slen;
    }
    /* We need space for both the length of the previous entry and
     * the length of the payload. */
    reqlen += zipStorePrevEntryLength(NULL,prevlen);
    reqlen += zipStoreEntryEncoding(NULL,encoding,slen);

    /* When the insert position is not equal to the tail, we need to
     * make sure that the next entry can hold this entry's length in
     * its prevlen field. */
    int forcelarge = 0;
    nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
    if (nextdiff == -4 && reqlen < 4) {
        nextdiff = 0;
        forcelarge = 1;
    }

    /* Store offset because a realloc may change the address of zl. */
    offset = p-zl;
    zl = ziplistResize(zl,curlen+reqlen+nextdiff);
    p = zl+offset;

    /* Apply memory move when necessary and update tail offset. */
    if (p[0] != ZIP_END) {
        /* Subtract one because of the ZIP_END bytes */
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

        /* Encode this entry's raw length in the next entry. */
        if (forcelarge)
            zipStorePrevEntryLengthLarge(p+reqlen,reqlen);
        else
            zipStorePrevEntryLength(p+reqlen,reqlen);

        /* Update offset for tail */
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

        /* When the tail contains more than one entry, we need to take
         * "nextdiff" in account as well. Otherwise, a change in the
         * size of prevlen doesn't have an effect on the *tail* offset. */
        zipEntry(p+reqlen, &tail);
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else {
        /* This element will be the new tail. */
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }

    /* When nextdiff != 0, the raw length of the next entry has changed, so
     * we need to cascade the update throughout the ziplist */
    if (nextdiff != 0) {
        offset = p-zl;
        zl = __ziplistCascadeUpdate(zl,p+reqlen);
        p = zl+offset;
    }

    /* Write the entry */
    p += zipStorePrevEntryLength(p,prevlen);
    p += zipStoreEntryEncoding(p,encoding,slen);
    if (ZIP_IS_STR(encoding)) {
        memcpy(p,s,slen);
    } else {
        zipSaveInteger(p,value,encoding);
    }
    ZIPLIST_INCR_LENGTH(zl,1);
    return zl;
}
```

