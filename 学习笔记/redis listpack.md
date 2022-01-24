### ziplist的缺点

1、不能存放过多元素，否则访问性能会下降
2、不能保存过大元素，否则引起内存重新分配，进而连锁更新

### listpack数据结构

```c
<listpack总字节数> <listpack总元素个数> <listpack entry> <listpack entry>... <listpack结束符255>
```

其中，entry的结构是：

```c
<entry-encoding> <entry-data> <entry-len>
编码类型  数据  编码类型+数据的总长度
```
