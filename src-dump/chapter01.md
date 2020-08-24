# Charpter01

源码以lua5.3.3进行分析。更多版本源码下载请访问官网 https://www.lua.org/ftp/ 

https://github.com/lymenglei/lua53-codedump

目录：

1. 基本的对象模型
2. 介绍字符串的存储
3. 介绍table的存储（比较多）
4. 介绍一点内存释放（gc）相关
5. 函数 TODO
6. 闭包 TODO
7. 编译 TODO
8. 其它 TODO

[toc]


## lua对象模型的基础介绍

Lua 8种数据类型

```
nil （以下5种是以值的形式存在）
boolean 
number 
userdata （full userdata）
lightuserdata (它不算lua的基本类型)

string (下面4中在vm中以引用方式共享)
table 
function 
thread 
```
下面介绍些lua的C源码中，常见的数据结构

#### GCObject Value TValue

lobject.h 72行，迎来了首个比较重要的数据结构`GCObject`，另外`CommonHeader`中的tt字段，可以理解为是用来标记lua 8种数据类型，故：
```c
/*
** Common Header for all collectable objects (in macro form, to be
** included in other objects)
*/
// 所有需要GC操作的数据都会加一个CommonHeader类型的宏定义
// next指向下一个GC链表的数据
// tt代表数据的类型以及扩展类型以及GC位的标志
// marked是执行GC的标记为，用于具体的GC算法
#define CommonHeader	GCObject *next; lu_byte tt; lu_byte marked

struct GCObject {
  CommonHeader;
};

// 展开之后
struct GCObject {
    GCObject * next;
    lu_byte tt;
    lu_byte marked;
}
```


另外一个非常重要的数据结构，`Value`
```c
typedef union Value {
  GCObject *gc;    /* collectable objects */
  void *p;         /* light userdata */
  int b;           /* booleans */
  lua_CFunction f; /* light C functions */
  lua_Integer i;   /* integer numbers */
  lua_Number n;    /* float numbers */
} Value;
```
Value类型是一个联合体，其中gc指针，把所有的GCObject串起来，这个在后面垃圾回收（gc）的时候会用到
其中p字段表示的是light userdata，下面介绍下light userdata 和 userdata的区别：

```
Userdata represent C values in Lua. A light userdata represents a pointer . It is a value (like a number)  
```
上面这句话是 lua 官网对二者的描述，从这里能看出来 light userdata实际上就是一个C指针，它的生命周期由C/C++控制，而userdata可以理解为，C/C++对象，创建是在lua里创建，内存的释放也是lua虚拟机来进行管理的。

接着上面的`Value`联合体说，b字段用来标示bool值，f字段是跟lua51版本比，新增加出来的一个字段

i字段，用来标示整型，n字段表示浮点数。（lua51中，只有一个lua_Number字段，53版本进行了一个细化拆分）


再往下面看，`TValue`类型
```c
typedef struct lua_TValue
{
    Value value_;
    int tt_;
}TValue;
```
这里在Value基础上，增加了tt_字段，来标示数据是什么类型，基础类型定义在lua.h头文件中




后面定义了一些常用的宏
`ttisXXX`表示判断这个o对象，是否是XXX类型

略过一段，在往下看
`setXXXvalue`宏，设置一个obj对象为XXX类型，并且赋值x
从这些宏就可以看出来，是如何设置一个TValue对象的类型，和对它进行赋值操作的是TValue的那个字段。


其它类型，string table Closure(闭包) Proto 暂时先不介绍，等到后面具体到某个类型在介绍。先了解这么多就可以了，因为后面忘了还是会回来重新看这些宏定义的。

#### nil值
参考chapter03，`查找key`


#### lua_State

另外还需要了解一个概念，就是`lua_State`，这是lua里的一个线程。定义在lstate.h
这里面有个`global_State`对象，理解为全局量都保存在这里。
后面介绍的 字符串 全部都保存在global_State的strt这个hash表中


#### 内存分配函数
在global_State中，有个字段
```c
lua_Alloc frealloc;  /* function to reallocate memory */
```
这是一个函数指针。
在strt的resize以及table的resize函数里，都会有看到类似的代码，去申请一块内存空间。
最终都会调用到如下部分：
```c
/*
** generic allocation routine.
*/
void *luaM_realloc_ (lua_State *L, void *block, size_t osize, size_t nsize) {
  void *newblock;
  global_State *g = G(L);
  size_t realosize = (block) ? osize : 0;
  lua_assert((realosize == 0) == (block == NULL));
#if defined(HARDMEMTESTS)
  if (nsize > realosize && g->gcrunning)
    luaC_fullgc(L, 1);  /* force a GC whenever possible */
#endif
  newblock = (*g->frealloc)(g->ud, block, osize, nsize);
  if (newblock == NULL && nsize > 0) {
    lua_assert(nsize > realosize);  /* cannot fail when shrinking a block */
    if (g->version) {  /* is state fully built? */
      luaC_fullgc(L, 1);  /* try to free some memory... */
      newblock = (*g->frealloc)(g->ud, block, osize, nsize);  /* try again */
    }
    if (newblock == NULL)
      luaD_throw(L, LUA_ERRMEM);
  }
  lua_assert((nsize == 0) == (newblock == NULL));
  g->GCdebt = (g->GCdebt + nsize) - realosize;
  return newblock;
}
```

搜来搜去，最后找到了这个。这个函数是lua提供的默认内存管理函数。
```c
static void *l_alloc (void *ud, void *ptr, size_t osize, size_t nsize) {
  (void)ud; (void)osize;  /* not used */
  if (nsize == 0) {
    free(ptr);
    return NULL;
  }
  else
    return realloc(ptr, nsize);
}
```

下面这段摘取自[博客](https://www.cnblogs.com/heartchord/p/4527494.html)
```txt
ud　 ：Lua默认内存管理器并未使用该参数。不过在用户自定义内存管理器中，可以让内存管理在不同的堆上进行。

ptr　：非NULL表示指向一个已分配的内存块指针，NULL表示将分配一块nsize大小的新内存块。

osize：原始内存块大小，默认内存管理器并未使用该参数。Lua的设计强制在调用内存管理器函数时候需要给出原始内存块的大小信息，如果用户需要自定义一个高效的内存管理器，那么这个参数信息将十分重要。这是因为大多数的内存管理算法都需要为所管理的内存块加上一个cookie，里面存储了内存块尺寸的信息，以便在释放内存的时候能够获取到尺寸信息(譬如多级内存池回收内存操作)。而Lua内存管理器刻意在调用内存管理器时提供了这个信息，这样就不必额外存储这些cookie信息，这样在大量使用小内存块的环境中将可以节省不少的内存。另外在ptr传入NULL时，osize表示Lua对象类型（LUA_TNIL、LUA_TBOOLEAN、LUA_TTHREAD等等），这样内存管理器就可以知道当前在分配的对象的类型，从而可以针对它做一些统计或优化的工作。

nsize：新的内存块大小，特别地，在nsize为0时需要提供内存释放的功能。
```


返回的就是`realloc`这个指针，最后都会调用到这个函数。

```c
void *realloc (void *ptr, size_t new_size );
```

> `realloc`函数用于修改一个原先已经分配的内存块的大小，可以使一块内存的扩大或缩小。当起始空间的地址为空，即`*ptr = NULL`,则同malloc。当*`ptr`非空：若nuw_size < size,即缩小`*ptr`所指向的内存空间，该内存块尾部的部分内存被拿掉，剩余部分内存的原先内容依然保留；若nuw_size > size,即扩大`*ptr`所指向的内存空间，如果原先的内存尾部有足够的扩大空间，则直接在原先的内存块尾部新增内存，如果原先的内存尾部空间不足，或原先的内存块无法改变大小，realloc将重新分配另一块new_size大小的内存，并把原先那块内存的内容复制到新的内存块上。因此，使用realloc后就应该改用realloc返回的新指针。



------------------

- lua中的gc管理，是采用了改进的`三色标记法`，具体三色标记法是如何设计，和如何gc的，请参考Chapter04，有很多动图讲解的很清楚，所以后面代码中会看到很多的白灰黑三色的一些变量以及宏定义。


------------------------
#### 字节码
> luac.exe -l -p xxx.lua

来查看xxx.lua 编译的字节码是什么格式的，其中lua的源码如下：

```lua

local a = { key = 1}
a.key = nil
```


```
menglei@menglei-PC MINGW64 /d/software/lua-5.3.3_Win32_bin
$ ./luac.exe -l -p a.lua

main <a.lua:0,0> (4 instructions at 0045e840)
0+ params, 2 slots, 1 upvalue, 1 local, 3 constants, 0 functions
        1       [2]     NEWTABLE        0 0 1
        2       [2]     SETTABLE        0 -1 -2 ; "key" 1
        3       [3]     SETTABLE        0 -1 -3 ; "key" nil
        4       [3]     RETURN          0 1

```



---------------------
## 参考文章：

https://www.cnblogs.com/heartchord/p/4527494.html  Lua内存管理器规则
