## Redis源码学习-6.0.15


#### intset结构

```c
//intset.h
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```


#### 创建空intset

```c
//intset.c
#define INTSET_ENC_INT16 (sizeof(int16_t))

intset *intsetNew(void) {
    //intset结构体变量并分配空间
    intset *is = zmalloc(sizeof(intset));
    //编码为sizeof(int16_t)
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    //长度0
    is->length = 0;
    return is;
}
```

#### 插入元素

```c
//intset.c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    //解析出encoding
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;

    /* Upgrade encoding if necessary. If we need to upgrade, we know that
     * this value should be either appended (if > 0) or prepended (if < 0),
     * because it lies outside the range of existing values. */
    if (valenc > intrev32ifbe(is->encoding)) {
        /* This always succeeds, so we don't need to curry *success. */
        return intsetUpgradeAndAdd(is,value);
    } else {
        //查询该元素是否已存在
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }

        //不存在的话，长度+1
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        //指针移到末尾
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    //插入元素
    _intsetSet(is,pos,value);
    //长度+1
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

#### 查找元素

```c
//intset.c
uint8_t intsetFind(intset *is, int64_t value) {
    uint8_t valenc = _intsetValueEncoding(value);
    return valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,NULL);
}

static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    /* The value can never be found when the set is empty */
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
        /* Check for the case where we know we cannot find the value,
         * but do know the insert position. */
        if (value > _intsetGet(is,max)) {
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }

    while(max >= min) {
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else {
        if (pos) *pos = min;
        return 0;
    }
}
```


#### 移除元素

```c
//intset.c
intset *intsetRemove(intset *is, int64_t value, int *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 0;

    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
        uint32_t len = intrev32ifbe(is->length);

        /* We know we can delete */
        if (success) *success = 1;

        /* Overwrite value with tail and update length */
        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos);
        is = intsetResize(is,len-1);
        is->length = intrev32ifbe(len-1);
    }
    return is;
}
```


>link:https://blog.csdn.net/yangbodong22011/article/details/78671625
