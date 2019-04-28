#chapter02 string

这里介绍下字符串在lua中是如何存储的

lua中的字符串全部是引用，相同的字符串只有一份存储。
string这部分相对比较独立，可以先拿出来讲一下。主要在lstring.h lstring.c 这两个文件中。

lua中字符串数据结构的定义 在文件lobject中，
```c
typedef struct TString {
  CommonHeader;
  lu_byte extra;  /* reserved words for short strings; "has hash" for longs */
  lu_byte shrlen;  /* length for short strings */
  unsigned int hash;
  union {
    size_t lnglen;  /* length for long strings */
    struct TString *hnext;  /* linked list for hash table */
  } u;
} TString;
```
从这个数据结构中，我们可以看出来lua是如何来存储字符串的：
- `CommonHeader` 标示这是一个需要GC的对象
- `extra` 用于记录辅助信息。对于短字符串，该字段用来标记字符串是否为保留字，用于词法分析器中对保留字的快速判断；对于长字符串，该字段将用于惰性求哈希值的策略（第一次用到才进行哈希）。
- `strlen` 字符串长度，lua里的字符串并不是以\0结尾的
- `hash` 哈希值
- `hnext` 哈希表中将所有相同hash值的字符串串成一个链表，该字段为下一个节点的指针




![lua string](./pic/c02_01.png)

lua 字符串在内存中的表示如上图





## 参考文章：
https://www.cnblogs.com/heartchord/p/4561308.html


