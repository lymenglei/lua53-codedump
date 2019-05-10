# Chapter04

https://github.com/lymenglei/lua53-codedump

[toc]


```txt
nil 只有全局一个引用吗
那些值会放到root set中
```


## GC Garbage Collect

`iscollectable`这个宏，用来检查一个TValue对象是否被标记为可以回收

```c
/* raw type tag of a TValue */
#define rttype(o)	((o)->tt_)

// 这个是看tag的第六位是不是1，是1的话就属于垃圾回收，否则就不需要关心它的生命周期
/* Bit mark for collectable types */
#define BIT_ISCOLLECTABLE	(1 << 6)

#define iscollectable(o)	(rttype(o) & BIT_ISCOLLECTABLE)

// 检查obj的生存期
// iscollectable(obj)检查obj是否为GC对象
// righttt(obj)返回obj的tt_是否等于gc里面的tt
// isdead(obj)返回obj是否已经被清理
// 总而言之，返回true代表未被GC的和不需要GC的，返回false代表已经被GC了的
#define checkliveness(L,obj) \
	lua_longassert(!iscollectable(obj) || \
		(righttt(obj) && (L == NULL || !isdead(G(L),gcvalue(obj)))))
```


---------------
## 参考文章：
https://github.com/lichuang/Lua-Source-Internal
