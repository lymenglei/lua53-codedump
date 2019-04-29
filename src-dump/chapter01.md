# Charpter01

联系：ly-menglei@163.com


目录：

1. 基本的对象模型
2. 介绍字符串的存储
3. 介绍table的存储（比较多）
4. 介绍一点内存释放（gc）相关
5. 函数 TODO
6. 闭包 TODO
7. 编译 TODO
8. 其它 TODO




## lua对象模型的基础介绍

lobject.h

#### GCObject Value TValue

72行，迎来了首个比较重要的数据结构`GCObject`，另外`CommonHeader`中的tt字段，可以理解为是用来标记lua 8种数据类型，故：
```c
// 展开之后
struct GCObject {
    GCObject * next;
    lu_byte tt;
    lu_byte marked;
}
```


下面迎来了另外一个非常重要的数据结构，`Value`
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
从英文的注释也能看出来，Value类型是一个联合体，其中gc指针，把所有的GCObject串起来，这个在后面垃圾回收（gc）的时候会用到
其中p字段表示的是light userdata，下面插播介绍下light userdata 和 userdata的区别：

```
Userdata represent C values in Lua. A light userdata represents a pointer . It is a value (like a number)  
```
上面这句话是 lua 官网对二者的描述，从这里能看出来 light userdata实际上就是一个C指针，它的生命周期有C/C++控制，而userdata可以理解为，C/C++对象，创建是在lua里创建，内存的释放也是lua虚拟机来进行管理的。

（TODO）接着上面的`Value`联合体说，b字段用来标示bool值，f字段是跟lua51版本比，新增加出来的一个字段

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

#### lua_State

另外还需要了解一个概念，就是`lua_State`，这是lua里的一个线程。定义在lstate.h
这里面有个`global_State`对象，理解为全局量都保存在这里。
后面介绍的 字符串 全部都保存在global_State的strt这个hash表中



------------------

- lua中的gc管理，是采用了改进的`三色标记法`，具体三色标记法是如何设计，和如何gc的，请自行百度搜索，有很多动图讲解的很清楚，所以后面代码中会看到很多的白灰黑三色的一些变量以及宏定义。

