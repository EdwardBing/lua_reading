lua5.3.5 源码学习
### 目录
1. lua基本数据类型介绍
2. lua table设计与实现
3. string的设计与实现
4. lua GC算法

##### string [参考博客](https://www.cnblogs.com/heartchord/p/4561308.html)
1. lua字符串内部被分为短字符串和长字符串，长字符串是为了处理http长文本所作出的优化。
字符串一经被创建就不可改写，Lua的值对象若为字符串类型，则它将以引用方式存在
	- 同样的string的数据结构也是被定义在了lobject.h里面
	```
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
	- CommonHeader 用于GC的对象
	- extra 用于记录辅助信息。对于短字符串，该字段用来标记字符串是否为保留字，用于词法分析器中对保留字的快速判断；对于长字符串，该字段将用于惰性求哈希值的策略（第一次用到才进行哈希）。
	- hash 记录字符串的哈希值，可以用来快速的查找和匹配。
	- shrlen 用于记录短字符串的长度。
	- 结构体u 里面lnglen记录长字符的长度。 hnext域用来指向 hash table 中相同哈希值的下一个字符串。
	- TString 结构图如下：
	![lua string](../pic/c02_01.png)
		- lua字符串对象 = TString + 实际字符串数据
		- TString结构 = GCObject指针 + 字符串信息数据
2. 短字符串与长字符串
	- lua字符串内建类型定义在lua.h中
	```
	// lua.h
	#define LUA_TSTRING		4
	```
	- 对于长短字符串来说，Lua在 LUA_TSTRING的宏上扩展了两个小类型
	```
	//lobject.h
	/* Variant tags for strings */
	#define LUA_TSHRSTR	(LUA_TSTRING | (0 << 4))  /* short strings */
	#define LUA_TLNGSTR	(LUA_TSTRING | (1 << 4))  /* long strings */
	```
	- 对短字符串的限制，在Lua的设计中，元方法名和保留字必须是短字符串，所以短字符串长度不得短于最长的元方法__newindex和保留字function的长度，也就是说LUAI_MAXSHORTLEN最小不可以设置低于10（字节）。
	```
	// llimits.h
	/*
	** Maximum length for short strings, that is, strings that are
	** internalized. (Cannot be smaller than reserved words or tags for
	** metamethods, as these strings must be internalized;
	** #("function") = 8, #("__newindex") = 10.)
	*/
	#if !defined(LUAI_MAXSHORTLEN)
	#define LUAI_MAXSHORTLEN	40
	#endif
	```
3. 字符串的创建查找流程
	![string init](../pic/c02_02.png)
	- luaS_new
	- 在lua中，字符串是被内化的一种数据结构，内化的意思就是说，每个存放lua字符串的变量，实际上存放的并不是一份字符串的数据副本，而是这份字符串的引用，因此，在lua中字符串是一个不可变的数据，然后呐，为了实现内化，在lua虚拟机中必然要存在一个全局的地方存放当前系统中的所有字符串，lua虚拟机使用一个散列通来管理字符串, global_State -> stringtable strt，结构图如下：
	![stringtable](../pic/c02_03.png)
	```
	TString *luaS_new (lua_State *L, const char *str) {
	  // 创建或重用一个以'\0'结尾的字符串，首先检查缓存(使用字符串的地址做为key来获取字符串)
	  unsigned int i = point2uint(str) % STRCACHE_N;  /* hash */
	  // 当创建一个字符串时，首先根据哈希算法，算出哈希值，这个算出来的哈希值就是strt数组的索引值，如果这个地方已经有值了，则使用链表串接起来（串接部分不能超过STRCACHE_M）
	  int j;
	  TString **p = G(L)->strcache[i];
	  for (j = 0; j < STRCACHE_M; j++) {
  		// 这个缓存能够包含以'\0'结尾的字符串，因此使用'strcmp'来检查是否命中是安全的
  		// 在lua中，字符串是被内化的一种数据结构，内化的意思就是说，每个存放lua字符串的变量，实际上存放的并不是一份字符串的数据副本，而是这份字符串的引用
	    if (strcmp(str, getstr(p[j])) == 0)  /* hit? */
	    // p指向的是strt的第一个链表，然后再比较str和当前的字符串有没有一样的，有的话，直接返回
	      return p[j];  /* that is it */
	  }
	  /* normal route */
	  // 首先要做的操作就是把当前strt中的元素，全部往后移一个位置，p[0]用来存放新建的字符串
	  for (j = STRCACHE_M - 1; j > 0; j--)
	    p[j] = p[j - 1];  /* move out last element */
	  /* new element is first in the list */
	  p[0] = luaS_newlstr(L, str, strlen(str));
	  return p[0];
	}
	```
