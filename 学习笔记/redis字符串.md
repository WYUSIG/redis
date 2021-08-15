## Redis源码学习



### 原C语言字符串缺点

char *s = "redis";

r e d i s \0

- 不保存长度，使用strlen()获取长度需要遍历字符串到\0，时间复杂度o(n)
- 字符串中间不能保存\0，不符合redis可以保存任意二进制数的定位
- 字符串拼接等都需要遍历到字符串末尾再拼接，时间效率低
- 拼接添加字符串时需要判断目标字符串是否有足够空间，否则需要程序员手动分配空间



### Redis SDS

SDS数据结构

- len：字符串数组现有长度
- alloc：字符串数组的分配空间长度
- flags：SDS类型
- buf[]：字符串数组

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
    fp = ((unsigned char*)s)-1;
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