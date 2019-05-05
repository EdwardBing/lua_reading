lua 源码学习
#### 2019.01.26
1. 使用lua语言这么长时间都没有去好好的品味lua的实现，实属惭愧啊。
	- 先回顾C语言基本语法，然后看云风大大lua源码赏析，循序渐进的学习。

#### 2019.4.18
1. 搁置了好久，重新拾起来阅读lua源码，期间哪里不懂就去网上翻阅云风大神的blog看。
	- C_Learn [union的使用和初始化](http://c.biancheng.net/view/375.html)
		- union区别于typedef最大的一点是union所有的成员在内存中从同一地址开始。
2. 开始拜读云风大神的对lua GC剖析
	- [云风lua GC](https://blog.codingnow.com/2011/04/lua_gc_multithreading.html)
	- [几种GC策略](https://blog.51cto.com/13914991/2294254)

#### 2019.4.19
1. 笔记
	- lua中有九种数据类型，而其中string table function thread四种是在VM中以引用共享，是需要被GC管理回收的对象。 还有proto和upvalue
	- 所有的GCObject都是以union+value的形式来保存的。并且所有的GCObject都是用一个单向链表串起来了。
	- string是单独管理，为了保证不会有重复的，string被放到一张巨大的hash表中管理。

#### 2019.4.24
1. 笔记 lua中链接多个版本库是造成lua vm的bug[lua 多个链接库问题](https://blog.codingnow.com/2012/01/lua_link_bug.html)
	- 在lua table实现中，为了不让hash table为空的时候不引用NULL指针（做的一种优化），引用了dummynode，这样就减少了操作表时的判断。 因为这个dummynode是静态分配出来的特殊节点，所以不能调用内存管理函数来释放它。当进程链接多份lua库的时候，就会出现多份的dummynode对象。
		```
			#define isdummy(n) ((n)==dummynode))
		```
2. string
	- 字符串保存子在状态机中有两种形式：短字符和长字符
	- Hash Dos 为了解决这个问题，把长字符串独立出来。大量文本处理中的输入的字符串不再通过哈希内部化进入全局字符串表。同时，使用了一个随机种子用于哈希值的计算，使攻击者无法轻易构造出拥有相同哈希值的不同字符串。

#### 2019.4.26
##### 两个链接
- [lua table详细讲解](https://www.cnblogs.com/herenzhiming/articles/5802011.html)
- [lua table设计与实现](https://www.cnblogs.com/zblade/p/8819609.html)
- [blog](https://manistein.github.io/blog/)

1. lua中的数据结构都为table是lua的一大特色，但是lua源码的实现中又把table分为数组部分和哈希表部分，数组部分是从1开始作整数索引。而不被数组部分存储的都放到哈希部分。
	```
	typedef struct Table {
	  CommonHeader;
	  lu_byte flags;  /* 1<<p means tagmethod(p) is not present */
	  lu_byte lsizenode;  /* log2 of size of `node' array */
	  struct Table *metatable;
	  TValue *array;  /* array part */
	  Node *node;
	  Node *lastfree;  /* any free position is before this position */
	  GCObject *gclist;
	  int sizearray;  /* size of `array' array */
	} Table;
	```
	- array是存储数组部分，sizearray则是数组长度。
	- node是存储的哈希部分，哈希表的大小是用lsizenode来表示，由于哈希表的大小一定为2的整数次幂，所以这里的lsizenode表示的是次幂，而不是哈希表的实际大小。
2. table的赋值：
	```
	TValue *luaH_set (lua_State *L, Table *t, const TValue *key) {
	  const TValue *p = luaH_get(t, key);
	  t->flags = 0;
	  if (p != luaO_nilobject)
	    return cast(TValue *, p);
	  else {
	    if (ttisnil(key)) luaG_runerror(L, "table index is nil");
	    else if (ttisnumber(key) && luai_numisnan(nvalue(key)))
	      luaG_runerror(L, "table index is NaN");
	    return newkey(L, t, key);
	  }
	}
	```
	- 它首先查找key是否在table中，若在，则直接替换原来的值，否则调用luaH_newkey，插入新的（key,value）。

	- 函数luaH_newkey代码如下（ltable.c）：
 	```
	TValue *luaH_newkey (lua_State *L, Table *t, const TValue *key) {
	  Node *mp;
	  if (ttisnil(key)) luaG_runerror(L, "table index is nil");
	  else if (ttisnumber(key) && luai_numisnan(L, nvalue(key)))
	    luaG_runerror(L, "table index is NaN");
	  mp = mainposition(t, key);
	  if (!ttisnil(gval(mp)) || isdummy(mp)) {  /* main position is taken? */
	    Node *othern;
	    Node *n = getfreepos(t);  /* get a free place */
	    if (n == NULL) {  /* cannot find a free place? */
	      rehash(L, t, key);  /* grow table */
	      /* whatever called 'newkey' take care of TM cache and GC barrier */
	      return luaH_set(L, t, key);  /* insert key into grown table */
	    }
	    lua_assert(!isdummy(n));
	    othern = mainposition(t, gkey(mp));
	    if (othern != mp) {  /* is colliding node out of its main position? */
	      /* yes; move colliding node into free position */
	      while (gnext(othern) != mp) othern = gnext(othern);  /* find previous */
	      gnext(othern) = n;  /* redo the chain with `n' in place of `mp' */
	      *n = *mp;  /* copy colliding node into free pos. (mp->next also goes) */
	      gnext(mp) = NULL;  /* now `mp' is free */
	      setnilvalue(gval(mp));
	    }
	    else {  /* colliding node is in its own main position */
	      /* new node will go into free position */
	      gnext(n) = gnext(mp);  /* chain new position */
	      gnext(mp) = n;
	      mp = n;
	    }
	  }
	  setobj2t(L, gkey(mp), key);
	  luaC_barrierback(L, obj2gco(t), key);
	  lua_assert(ttisnil(gval(mp)));
	  return gval(mp);
	}
	```
	- 往table中插入新的值，其基本思路是检测key的主位置（main position）是否为空，这里主位置就是key的哈希值在node数组中（哈希表）的位置。若主位置为空，则直接把相应的（key,value）插入 到这个node中。若主位置被占了，检查占领该位置的（key,value）的主位置是不是在这个地方，若不在这个地方，则移动占领该位置的 （key,value）到一个新的空node中，并且把要插入的（key,value）插入到相应的主位置；若在这个地方（即占领该位置的 （key,value）的主位置就是要插入的位置），则把要插入的（key,value）插入到一个新的，空node中。若找不到空闲位置放置新键值，则 进行rehash函数，扩增加或减少哈希表的大小找出新位置，然后再调用luaH_set把要插入的（key,value）到新的哈希表中，直接返回 LuaH_set的结果。
	- rehash首先统计当前table中到底有value值不是nil的键值对的个数，然后根据这个数值确定table中数组部分的大小（其大小保证数组部分的空间利用率必须50%），最后调用luaH_resize函数来重建table。 
	具体过程是首先调用函数numusearray计算table中数组部分非nil的数值的个数，然后调用numusehash函数计算table中哈希部分的非nil的键值对的个数。调用countint函数来确定将要插入的（key,value）是否可以放在数组部分，接着调用computesizes来计算新的table数组部分的大小，最后调用luaH_resize函数根据原来table中数据构建新的table。
	```
	static void rehash (lua_State *L, Table *t, const TValue *ek) {
	  int nasize, na;
	  int nums[MAXBITS+1];  /* nums[i] = number of keys between 2^(i-1) and 2^i */
	  int i;
	  int totaluse;
	  for (i=0; i<=MAXBITS; i++) nums[i] = 0;  /* reset counts */
	  nasize = numusearray(t, nums);  /* count keys in array part */
	  totaluse = nasize;  /* all those keys are integer keys */
	  totaluse += numusehash(t, nums, &nasize);  /* count keys in hash part */
	  /* count extra key */
	  nasize += countint(ek, nums);
	  totaluse++;
	  /* compute new size for array part */
	  na = computesizes(nums, &nasize);
	  /* resize the table to new computed sizes */
	  resize(L, t, nasize, totaluse - na);
	}
	```

3. 总结：
	- 在对table操作时，尽量不要触发rehash操作，因为这个开销是非常大的。在对table插入新的键值对时（也就是说key原来不在 table中），可能会触发rehash操作，而直接修改已存在key对于的值，不会触发rehash操作的，包括赋值为nil。
	- 在遍历一个table时，不允许向table插入一个新键，否则将无法预测后续的遍历行为，但lua允许在遍历过程中，修改table中已存在的键对应的值，包括修改后的值为nil，也是允许的。
	- table中要想删除一个元素等同于向对应key赋值为nil，等待垃圾回收。但是删除table一个元素时候，并不会触发表重构行为，即不会触发rehash操作。
	- 为了减少rehash操作，当构造一个数组时，如果预先知道其大小，可以预分配数组大小。在脚本层可以使用local t = {nil,nil,nil}来预分配数组大小。
	- 注意在使用长度操作符#对数组其长度时，数组不应该包含nil值，否则很容易出错。