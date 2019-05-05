lua5.3.5 源码学习
### 目录
1. lua基本数据类型介绍
2. lua table设计与实现
3. string的设计与实现
4. lua GC算法

##### lua基本数据类型简述
1. 首先lua介绍lua的8种基本数据类型：
	- 定义在lua.h
	```
		#define LUA_TNIL		0
		#define LUA_TBOOLEAN		1
		#define LUA_TLIGHTUSERDATA	2
		#define LUA_TNUMBER		3
		#define LUA_TSTRING		4
		#define LUA_TTABLE		5
		#define LUA_TFUNCTION		6
		#define LUA_TUSERDATA		7
		#define LUA_TTHREAD		8
		#define LUA_NUMTAGS		9
	```
	- 数据对象定义在lobject.h, 可以看出TValue是一个结构体，里面是TValuefields，由一个宏定义构成。TValuefields包含Value和一个int类型的值，这里的tt_比较有意思存放lua的数据类型。而Value是一个联合体，关于联合体这里不做阐述。
	Value中的由一个gc指针，这里是被GC管理的对象，这个以后学习GC算法的时候在概述。
	p是light userdata的指针。i,n是int和float类型的numbers。
		- 这里要说明下[union的使用和初始化](http://c.biancheng.net/view/375.html)
		- union区别于typedef最大的一点是union所有的成员在内存中从同一地址开始
	```
	/*
	** Union of all Lua values
	*/
	typedef union Value {
	  GCObject *gc;    /* collectable objects */
	  void *p;         /* light userdata */
	  int b;           /* booleans */
	  lua_CFunction f; /* light C functions */
	  lua_Integer i;   /* integer numbers */
	  lua_Number n;    /* float numbers */
	} Value;
	#define TValuefields	Value value_; int tt_
	typedef struct lua_TValue {
	  TValuefields;
	} TValue;
	```
##### Table [ltable.c解析](https://blog.csdn.net/u013517637/article/details/78899279)
1. 我们知道在lua中一切数据结构皆是table，这是lua语言的一大特色。但是lua的源码实为了效率，把table分为了两部分即数组部分和哈希部分。
	Table的内部数据结构被定义在了lobject.h中
	```
	typedef struct Table {
	  CommonHeader;
	  lu_byte flags;  /* 1<<p means tagmethod(p) is not present */
	  lu_byte lsizenode;  /* log2 of size of 'node' array */
	  unsigned int sizearray;  /* size of 'array' array */
	  TValue *array;  /* array part */
	  Node *node;
	  Node *lastfree;  /* any free position is before this position */
	  struct Table *metatable;
	  GCObject *gclist;
	} Table;
	```
	这里做几个说明：
	- CommonHeader 为所有GC可回收的对象提供标记头。所有需要GC操作的数据都会加一个 next指向下一个GC链表的数据tt代表数据的类型以及扩展类型以及GC位的标志
	marked是执行GC的标记为，用于具体的GC算法
		```
			#define CommonHeader	GCObject *next; lu_byte tt; lu_byte marked
		```
	- lua_byte flags 这是一个lua的byte类型的数据，用于表示表中提供了哪些元方法，比如是否提供了元方法_index，该数据最开始设置为1，如果进行查找一次，比如_index，如果存在，这该元方法对应的flag bit设置为0，在下一次查找的时候，只需要比较这个bit即可，对应的元方法在ltm.h中
		```
		typedef unsigned char lu_byte;
		```
	- lsizenode 为散列表的大小，必定为2的幂对应的数字
	- metatable 该table的元表
	- array 该table的数组的指针
	- node 该table的散列表的起始位置的指针
	- lastfree 该散列表的最后位置的指针
	- gclist gc相关的链表
	- sizearray 数组的大小，不一定为2的幂对应的数字
	- 对于node数据，类似于其他语言中的字典设计\hash设计，就是一个键值对集合，其定义为：
		- 需要提一下的是对于key的设计采用的是union，也就是说Lua的散列表的key，可以为nk对应的struct，也可以是TValue类型
	```
	// 每个节点都有key和val
	typedef struct Node {
	  TValue i_val;
	  TKey i_key;
	} Node;
	//TKey
	typedef union TKey {
	  struct {
	    TValuefields;
	    int next;  /* for chaining (offset for next node) */
	  } nk;
	  TValue tvk;
	} TKey;
	```
	- table的创建 luaH_new
	```
	// luaH_new创建的Table开始实际上是一张空表
	// luaC_newobj作用是创建一个可被回收的对象（需要提供类型和大小），并且被链接到gc列表中
	// gco2t宏的作用是把gc对象转换为table，内部操作为把o转化为GCUnion之后，再取h（代表table）的地址
	// 调用setnodevector()为表哈希节点项分配内存并初始化
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
	- table新增元素的实现，给table中新加一个元素很可能会触发数组部分和哈希部分的重新分配。其本质是rehash（但是对于下表超过数组长度的数字，都会放到哈希部分，所以数组的插值不会触发rehash）。
	```
	TValue *luaH_newkey (lua_State *L, Table *t, const TValue *key) {
	  Node *mp;
	  TValue aux;
	  // 在插入一个新的key的时候首先判断key是不是NULL，是的话就报错
	  if (ttisnil(key)) luaG_runerror(L, "table index is nil");
	  else if (ttisfloat(key)) {
	    lua_Integer k;
	    //luaV_tointeger 看一下 key的索引是否可以转换为int，此处0的意思就是不接受取整，只能是int
	    if (luaV_tointeger(key, &k, 0)) {  /* does index fit in an integer? */
	      // 然后用构造出来的aux对key初始化，这样就算是处理完了key，在拿着key来取得最重要的mainposition
	      setivalue(&aux, k);
	      key = &aux;  /* insert it as an integer */
	    }
	    else if (luai_numisnan(fltvalue(key)))
	      // 然后接着判断，如果是数字，若是未定义数字也错误返回
	      luaG_runerror(L, "table index is NaN");
	  }
	  // 然后调用mainposition求出t中key对应的mainposition，返回值是一个Node *类型
	  mp = mainposition(t, key);
	  // 如果t的哈希部分为NULL或者mp节点不为NULL都会走if的逻辑
	  if (!ttisnil(gval(mp)) || isdummy(t)) {  /* main position is taken? */
	    // 首先就是定义一个othern的节点和一个f节点
	    Node *othern;
	    // 尝试获取一个空闲位置
	    Node *f = getfreepos(t);  /* get a free place */
	    // getfreepos获取一个可以用的空闲节点,如果返回NULL则说明哈希部分满员了 需要重新哈希
	    if (f == NULL) {  /* cannot find a free place? */
	      rehash(L, t, key);  /* grow table hash表扩容*/
	      /* whatever called 'newkey' takes care of TM cache */
	      return luaH_set(L, t, key);  /* insert key into grown table */
	    }
	    lua_assert(!isdummy(t));
	    // 当返回一个可用的节点时，会判断:
	    othern = mainposition(t, gkey(mp));
	    //  如果不属于同一主位置节点链下，则意味着原本通过 mainposition(newkey) 直接定位的节点被其它节点链中的某个节点占用
	    if (othern != mp) {  /* is colliding node out of its main position? */
	      /* yes; move colliding node into free position */
	      // 如果得到的mp被其他节点链中的节点占用时，
	      // 则由othern开始开始向后遍历一直到mp的前一个位置，
	      // 然后将mp的数据移动到空节点上，
	      // othern指向新的节点，旧的节点用来放入新插入的键
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
	- 接下来说下table中比较重要的操作rehash
		- nums中存放的是元素的数量
		- 分表遍历数组(numusearray)和散列表(numusehash)，统计更新nums中的数量大小
		- 重新计算数组和hash部分的大小，数组大小的计算规则：逐个遍历nums数组，获得其范围区间内所包含的整数数量大于50%的最大索引，作为rehash后的数组大小，这个索引值来自与computesizes函数：
	```
	//对table进行重新划分hash和数组部分的大小
	static void rehash (lua_State *L, Table *t, const TValue *ek) {
	  unsigned int asize;  /* optimal size for array part */
	  unsigned int na;  /* number of keys in the array part */
	  // nums存放的是key在2^(i - 1)到2^i直接的元素数量
	  unsigned int nums[MAXABITS + 1];
	  int i;
	  int totaluse;
	  for (i = 0; i <= MAXABITS; i++) nums[i] = 0;  /* reset counts */
	  na = numusearray(t, nums);  /* count keys in array part */
	  totaluse = na;  /* all those keys are integer keys */
	  totaluse += numusehash(t, nums, &na);  /* count keys in hash part */
	  /* count extra key */
	  //判断k的范围
	  na += countint(ek, nums);
	  totaluse++;
	  /* compute new size for array part */
	  //计算数组部分的大小
	  asize = computesizes(nums, &na);
	  /* resize the table to new computed sizes */
	  //重新分配数组和哈希部分大小
	  luaH_resize(L, t, asize, totaluse - na);
	}
	```
	- 详细看下 computesizes 函数
		- 首先nums数组在统计后，每个下标对应的是处于当前2^(i -1) - 2^i中的元素的个数，然后不断的累加计算，求得满足 sum > 2^n/2的最大下标值(这个下标值是nums数组中的)
		- 所以，在不同的rehash阶段，table中的同一个key可能会在数组部分和散列表部分交替出现，也是可能的。
		- 由于rehash会带来较大的性能消耗，所以一般都尽量避免，比如在创建表的时候，就采用预填充的算法
	```
	/ 只有利用率超过50%的数组才会放到数组部分，否则放到哈希部分
	static unsigned int computesizes (unsigned int nums[], unsigned int *pna) {
	  int i;
	  unsigned int twotoi;  /* 2^i (candidate for optimal size) */
	  unsigned int a = 0;  /* number of elements smaller than 2^i */
	  unsigned int na = 0;  /* number of elements to go to array part */
	  unsigned int optimal = 0;  /* optimal size for array part 数组最优大小 */ 
	  /* loop while keys can fill more than half of total size */
	  // 循环遍历直到key可以填满数组部分大小的一半
	  for (i = 0, twotoi = 1;
	       twotoi > 0 && *pna > twotoi / 2;
	       i++, twotoi *= 2) {
	    if (nums[i] > 0) {
	      a += nums[i];
	      if (a > twotoi/2) {  /* more than half elements present? */
	        optimal = twotoi;  /* optimal size (till now) */
	        na = a;  /* all elements up to 'optimal' will go to array part */
	      }
	    }
	  }
	  lua_assert((optimal == 0 || optimal / 2 < na) && na <= optimal);
	  *pna = na;
	  return optimal;
	}
	```
##### string [lua字符串](https://www.cnblogs.com/heartchord/p/4561308.html)
- 