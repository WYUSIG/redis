### stream消息示例

#### 生产消息
```c
XADD devmsg * dev 3 temp 26
XADD devmsg * dev 4 temp 24
XADD devmsg * dev 5 temp 27
```

### stream数据结构选择

在stream消息中，会含有大量相同键值的消息，这时候你可能会想到用哈希表来存，但是因为有大量相同键值，这就导致冗余不少数据，在redis中，选择了前缀树+listpack的数据结构

```c
typedef struct stream {
    rax *rax;               /* The radix tree holding the stream. 前缀树*/
    uint64_t length;        /* Number of elements inside this stream. stream里面的消息数*/
    streamID last_id;       /* Zero if there are yet no items. 消息流中最后一个消息的id*/
    rax *cgroups;           /* Consumer groups dictionary: name -> streamCG 消费者组信息*/
} stream;

typedef struct rax {
    raxNode *head; //前缀树头节点
    uint64_t numele; //前缀树key个数
    uint64_t numnodes; //前缀树raxNode的个数
} rax;

typedef struct raxNode {
    uint32_t iskey:1;     /* Does this node contain a key? 节点是否包含key*/
    uint32_t isnull:1;    /* Associated value is NULL (don't store it). 节点是否为null*/
    uint32_t iscompr:1;   /* Node is compressed. 节点是否被压缩*/
    uint32_t size:29;     /* Number of children, or compressed string len. 节点大小*/
    unsigned char data[];  //节点的实际存储数据
} raxNode;
```