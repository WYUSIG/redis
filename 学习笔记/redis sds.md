## Redis源码学习-6.0.15



### 原C语言字符串缺点

char *s = "redis";

r e d i s \0

- 不保存长度，使用strlen()获取长度需要遍历字符串到\0，时间复杂度o(n)
- 字符串中间不能保存\0，不符合redis可以保存任意二进制数的定位
- 字符串拼接等都需要遍历到字符串末尾再拼接，时间效率低
- 拼接添加字符串时需要判断目标字符串是否有足够空间，否则需要程序员手动分配空间



### Redis SDS

>源码文件：sds.h、sds.c

SDS数据结构

- len：字符串数组现有长度
- alloc：字符串数组的分配空间长度
- flags：SDS类型
- buf[]：字符串数组

##### 代码定义
```c
/* __attribute__ ((__packed__))是希望编译器严格按照结构体长度分配空间，不要只申请3字节给8字节这种
 * 可以看到主要是len、alloc的类型不同，这是Redis针对不同长度的字符串定义不同类型长度的元数据字段，精打细算节省空间 */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

##### 定义char*别名

```c
typedef char *sds;
```

##### 创建sds字符串

```c
/**
 * 创建新的sds字符串
 * @param init 初始化字符串
 * @param initlen 初始化长度
 * @return sds
 */
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;  //定义一个指针
    sds s; //char*字符数组
    /**
    * Redis针对不同长度字符串定义不同的结构体，是为了节省空间
    */
    char type = sdsReqType(initlen); //根据初始字符串长度获取类型
    //如果是空字符串，又是最小类型SDS_TYPE_5，那就升级到SDS_TYPE_8，因为空字符串一般用来后面append
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    //该类型结构体长度
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */
	
    //防止长度相加后溢出
    assert(initlen + hdrlen + 1 > initlen); /* Catch size_t overflow */
    //给sh这个指针分配结构体长度+初始长度+1的空间
    sh = s_malloc(hdrlen+initlen+1);
    //分配空间失败
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT)
        //如果是“SDS_NOINIT”，init为不需要初始化
        init = NULL;
    else if (!init)
        //如果不需要初始化，全部填充0
        memset(sh, 0, hdrlen+initlen+1);
    s = (char*)sh+hdrlen;
    //s[-1]
    fp = ((unsigned char*)s)-1;
    //设置类型、总长度、分配长度
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }
    if (initlen && init)
        //如果初始字符串长度不为0，且需要初始化，则把init放进s中
        memcpy(s, init, initlen);
    //补充末尾0
    s[initlen] = '\0';
    return s;
}
```

##### 拼接字符串

```c
/* Append the specified binary-safe string pointed by 't' of 'len' bytes to the
 * end of the specified sds string 's'.
 *
 * After the call, the passed sds string is no longer valid and all the
 * references must be substituted with the new pointer returned by the call. */
sds sdscatlen(sds s, const void *t, size_t len) {
    //当前sds字符串长度
    size_t curlen = sdslen(s);

    //如有必要，进行扩容
    s = sdsMakeRoomFor(s,len);
    if (s == NULL) return NULL;
    //拼接目标字符串
    memcpy(s+curlen, t, len);
    //重新设置sds字符串长度
    sdssetlen(s, curlen+len);
    //末尾添加结束符
    s[curlen+len] = '\0';
    return s;
}
```

##### 扩容

```c
/* Enlarge the free space at the end of the sds string so that the caller
 * is sure that after calling this function can overwrite up to addlen
 * bytes after the end of the string, plus one more byte for nul term.
 *
 * Note: this does not change the *length* of the sds string as returned
 * by sdslen(), but only the free buffer space we have. */
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    //s的剩余可用长度
    size_t avail = sdsavail(s);
    size_t len, newlen;
    //sds类型
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;

    /* Return ASAP if there is enough space left. */
    //如果剩余可用长度大于新增的长度，不进行扩容
    if (avail >= addlen) return s;

    //s的总长度
    len = sdslen(s);
    //获取sds的字符串数据，去除元数据, 剩余char*真正字符串部分
    sh = (char*)s-sdsHdrSize(oldtype);
    newlen = (len+addlen);
    //防止相加溢出
    assert(newlen > len);   /* Catch size_t overflow */
    if (newlen < SDS_MAX_PREALLOC)
        //小于1024*1024，两倍扩容
        newlen *= 2;
    else
        //大于或等于1024*1024，扩容1M
        newlen += SDS_MAX_PREALLOC;

    //根据新总长度，获取sds类型
    type = sdsReqType(newlen);

    /* Don't use type 5: the user is appending to the string and type 5 is
     * not able to remember empty space, so sdsMakeRoomFor() must be called
     * at every appending operation. */
    //不推荐使用SDS_TYPE_5
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    //获取元数据长度
    hdrlen = sdsHdrSize(type);
    //防止溢出
    assert(hdrlen + newlen + 1 > len);  /* Catch size_t overflow */
    //类型不变
    if (oldtype==type) {
        //扩容
        newsh = s_realloc(sh, hdrlen+newlen+1);
        //判断是否扩容失败
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        /* Since the header size changes, need to move the string forward,
         * and can't use realloc */
        //扩容
        newsh = s_malloc(hdrlen+newlen+1);
        //判断是否扩容失败
        if (newsh == NULL) return NULL;
        //复制旧sds字符串数据部分
        memcpy((char*)newsh+hdrlen, s, len+1);
        //是否旧sds字符串
        s_free(sh);
        //加上新的元数据
        s = (char*)newsh+hdrlen;
        //设置类型
        s[-1] = type;
        //设置长度
        sdssetlen(s, len);
    }
    //设置sds的分配长度(alloc)
    sdssetalloc(s, newlen);
    return s;
}
```