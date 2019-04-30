# Chapter03

https://github.com/lymenglei/lua53-codedump

## table

在第一章介绍了TValue类型数据结构，在介绍table之前，需要介绍两个重要的类型：
```c
typedef union TKey {
  struct {
    TValuefields;
    int next;  /* for chaining (offset for next node) */
  } nk;
  TValue tvk;
} TKey;

typedef struct Node {
  TValue i_val;
  TKey i_key;
} Node;
```

对于TKey，任何时候只有两种类型，要么是整数，要么不是整数(not nil)
next字段在之前版本是指针，5.3版本换成了便宜，指向下一个偏移的节点

Node节点是table的节点值。

然后是真正的`table`类型的定义：

```c
typedef struct Table {
  CommonHeader;
  lu_byte flags;  /* 1<<p means tagmethod(p) is not present */
  lu_byte lsizenode;  /* log2 of size of 'node' array */
  unsigned int sizearray;  /* size of 'array' array */
  TValue *array;  /* array part */
  Node *node;  // hash部分
  Node *lastfree;  /* any free position is before this position */
  struct Table *metatable;
  GCObject *gclist;
} Table;
```

对于Table，先挑重点的说。
我们知道table内部实际上分为数组部分和hash部分，其中数组部分存在array数组里，hash部分存放在node数组里；数组部分的容量为sizearray，hash部分容量为2的lsizenode次幂（hash部分容量总是2的N次幂，这个规则后面还会提到），lastfree是一个指针，初始指向一个dummyNode，之后会随着插入新节点产生冲突时，由node数组的尾部向前移动。

根据lua代码中使用table的情况，会从构造一个table，索引，插入，删除等来分析table内部是如何存储值的。主要在ltable.c中

---------------------------

#### 构造一个空table

```c
Table *luaH_new (lua_State *L) {
  GCObject *o = luaC_newobj(L, LUA_TTABLE, sizeof(Table));
  Table *t = gco2t(o);
  t->metatable = NULL;
  t->flags = cast_byte(~0);
  t->array = NULL;
  t->sizearray = 0;
  setnodevector(L, t, 0);
  return t;
}
```
`luaH_new`函数是创建一个空的table，luaC_newobj函数定义在lgc.c中，在上一章，创建一个字符串对象`createstrobj`方法时候也有用到。


`gco2t`这个宏，将o的对象类型转换为Table，这里还要介绍下`GCUnion`

```c
/*
** Union of all collectable objects (only for conversions)
*/
union GCUnion {
  GCObject gc;  /* common header */
  struct TString ts;
  struct Udata u;
  union Closure cl;
  struct Table h;
  struct Proto p;
  struct lua_State th;  /* thread */
};
```

```c
#define gco2t(o)  check_exp((o)->tt == LUA_TTABLE, &((cast_u(o))->h))
```

gco2t最后会调用到cast_u这个宏
```c
#define cast_u(o)	cast(union GCUnion *, (o))
```

还有很多类似 `gco2XXX` 的宏，这里就不一一介绍了。

将new出来的table对象，metatable元表字段设置为空，数组部分初始为空，大小为0

hash部分，则通过`setnodevector`函数来调整。（这个函数在resize时候还会提到）
初始化时候size为0，node指向了一个dummynode，hash部分的size也是0，lastfree是个空指针。

-------------------------------

#### 向table中插入一个元素

这个函数比较长，下面慢慢说。其中一些简单的宏定义就不说明了，基本上lua源码里的宏都还算很好理解。

```c
TValue *luaH_newkey (lua_State *L, Table *t, const TValue *key) {
  Node *mp;
  TValue aux;
  if (ttisnil(key)) luaG_runerror(L, "table index is nil");
  else if (ttisfloat(key)) {
    lua_Integer k;
    if (luaV_tointeger(key, &k, 0)) {  /* does index fit in an integer? */
      setivalue(&aux, k);
      key = &aux;  /* insert it as an integer */
    }
    else if (luai_numisnan(fltvalue(key)))
      luaG_runerror(L, "table index is NaN");
  }
  mp = mainposition(t, key);
  if (!ttisnil(gval(mp)) || isdummy(t)) {  /* main position is taken? */
    Node *othern;
    Node *f = getfreepos(t);  /* get a free place */
    if (f == NULL) {  /* cannot find a free place? */
      rehash(L, t, key);  /* grow table */
      /* whatever called 'newkey' takes care of TM cache */
      return luaH_set(L, t, key);  /* insert key into grown table */
    }
    lua_assert(!isdummy(t));
    othern = mainposition(t, gkey(mp));
    if (othern != mp) {  /* is colliding node out of its main position? */
      /* yes; move colliding node into free position */
      while (othern + gnext(othern) != mp)  /* find previous */
        othern += gnext(othern);
      gnext(othern) = cast_int(f - othern);  /* rechain to point to 'f' */
      *f = *mp;  /* copy colliding node into free pos. (mp->next also goes) */
      if (gnext(mp) != 0) {
        gnext(f) += cast_int(mp - f);  /* correct 'next' */
        gnext(mp) = 0;  /* now 'mp' is free */
      }
      setnilvalue(gval(mp));
    }
    else {  /* colliding node is in its own main position */
      /* new node will go into free position */
      if (gnext(mp) != 0)
        gnext(f) = cast_int((mp + gnext(mp)) - f);  /* chain new position */
      else lua_assert(gnext(f) == 0);
      gnext(mp) = cast_int(f - mp);
      mp = f;
    }
  }
  setnodekey(L, &mp->i_key, key);
  luaC_barrierback(L, t, key);
  lua_assert(ttisnil(gval(mp)));
  return gval(mp);
}
```

有几个关键的函数，其中一个就是`mainposition`，可以理解为根据传进来的参数和它对应的类型，计算出来一个在hash数组里的Node地址，也就是在hash表中的位置。当然不同的key可能会有相同的Node*地址，这个时候就发生了冲突。

```c
static Node *mainposition (const Table *t, const TValue *key) {
  switch (ttype(key)) {
    case LUA_TNUMINT:
      return hashint(t, ivalue(key));
    case LUA_TNUMFLT:
      return hashmod(t, l_hashfloat(fltvalue(key)));
    case LUA_TSHRSTR:
      return hashstr(t, tsvalue(key));
    case LUA_TLNGSTR:
      return hashpow2(t, luaS_hashlongstr(tsvalue(key)));
    case LUA_TBOOLEAN:
      return hashboolean(t, bvalue(key));
    case LUA_TLIGHTUSERDATA:
      return hashpointer(t, pvalue(key));
    case LUA_TLCF:
      return hashpointer(t, fvalue(key));
    default:
      lua_assert(!ttisdeadkey(key));
      return hashpointer(t, gcvalue(key));
  }
}
```

这里将元素放入指定的hash[]位置时候，有一些原则。初始化的时候lastfree指向hash数组的最后一个指针；如果计算出来的mainposition没有元素，那么就把元素放在这个位置；若这个mainposition有元素，那么就向前移动lastfree，直到找到一个空的位置，将元素放在这里，并且设置一个next指针指向这里；还有一种情况，就是mainposition有元素，但是mainposition位置的元素计算出来的mainposition并不是这个位置，也就是说他是用链表连接起来的，那么就把这个元素向前找lastfree，然后把真正mainposition的元素插在这里。

这里描述的挺乱的，下面就引用一篇[博文](https://blog.csdn.net/fwb330198372/article/details/88579361)中的图片进行详细解释：


- 初始化时，向Table插入一个元素
    ```lua
    Table["k0"] = "v0"
    ```
    假设k0落在node[3]的位置，此时hash部分如下图：
![hash1](./pic/c03_1.png)

- 向Table插入第二个元素
    ```lua
    Table["k1"] = "v1"
    ```
    假设此时mainposition计算的节点与k0的节点相同，那么此时发生了冲突，向左移动lastfree指针，找到node[2]位置是个空的，所以讲k1节点放在node[2]，并且node[3]的next指向node[2]
![hash2](./pic/c03_2.png)

- 向Table插入第三个元素
    ```lua
    Table["k2"] = "v2"
    ```
    假设此时mainposition计算的节点与k1的节点相同，又冲突了，但是此时与上面的冲突稍有不同，此时k1节点并不是在它真正的mainposition位置，而k2的真正的mainposition是这个节点，那么就要做出优先级让步，移动k1这个节点，把k2放在k1的位置，同时修改next指向
![hash3](./pic/c03_3.png)

---------------------------

#### table的rehash

触发table 做rehash 操作的地方只有在向table中插入一个新key的时候，会调用`getfreepos`来寻找一个可用的位置，当这个函数返回空的时候，才进行rehash操作，也就是hash部分全部填满？？？
```c
static Node *getfreepos (Table *t) {
  if (!isdummy(t)) {
    while (t->lastfree > t->node) { // 从后面向前找位置
      t->lastfree--;
      if (ttisnil(gkey(t->lastfree)))
        return t->lastfree;
    }
  }
  return NULL;  /* could not find a free place */
}
```

rehash函数：

```c
static void rehash (lua_State *L, Table *t, const TValue *ek) {
  unsigned int asize;  /* optimal size for array part */
  unsigned int na;  /* number of keys in the array part */
  unsigned int nums[MAXABITS + 1];
  int i;
  int totaluse;
  for (i = 0; i <= MAXABITS; i++) nums[i] = 0;  /* reset counts */
  na = numusearray(t, nums);  /* count keys in array part */
  totaluse = na;  /* all those keys are integer keys */
  totaluse += numusehash(t, nums, &na);  /* count keys in hash part */
  /* count extra key */
  na += countint(ek, nums);
  totaluse++;
  /* compute new size for array part */
  asize = computesizes(nums, &na);
  /* resize the table to new computed sizes */
  luaH_resize(L, t, asize, totaluse - na);
}
```





--------------------

## 参考文章：
https://blog.csdn.net/fwb330198372/article/details/88579361
