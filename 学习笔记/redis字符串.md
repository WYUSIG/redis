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





```c
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;  //定义一个指针
    sds s;
    char type = sdsReqType(initlen);
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */

    assert(initlen + hdrlen + 1 > initlen); /* Catch size_t overflow */
    sh = s_malloc(hdrlen+initlen+1);
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
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
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    return s;
}
```