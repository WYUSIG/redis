### ziplist的缺点

1、不能存放过多元素，否则访问性能会下降
2、不能保存过大元素，否则引起内存重新分配，进而连锁更新

### quicklist数据结构

其实quicklist就是在ziplist上再封装了一层链表，防止单个ziplist元素过多，或者一个大元素进来会引起内存重新分配

```c
//quicklist.h
typedef struct quicklistNode {
    struct quicklistNode *prev; //前一个quicklistNode
    struct quicklistNode *next; //后一个quicklistNode
    unsigned char *zl;  //quicklistNode指向的ziplist
    unsigned int sz;             /* ziplist size in bytes ziplist字节个数*/
    unsigned int count : 16;     /* count of items in ziplist ziplist元素个数*/
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 编码方式：原生字节数组和压缩存储*/
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 存储方式*/
    unsigned int recompress : 1; /* was this node previous compressed? 数据是否被压缩*/
    unsigned int attempted_compress : 1; /* node can't compress; too small 数据能否被压缩*/
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;

typedef struct quicklist {
    quicklistNode *head;  //quicklist链表头
    quicklistNode *tail;  //quicklist链表尾
    unsigned long count;        /* total count of all entries in all ziplists quicklist的ziplist个数*/
    unsigned long len;          /* number of quicklistNodes quicklist的quicklistNode个数*/
    int fill : QL_FILL_BITS;              /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;
```