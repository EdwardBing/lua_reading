函数和闭包（function and Closure）

---

### 闭包
- 一个简单的闭包如下：
	```
	function makecounter()
		local t = 0
		return function()
			t = t + 1
			return t
		end
	end
	```
- 当调用 makecounter 后，会得到一个函数。这个函数每调用一次，返回值就会递增一。顾名思义，我们
可以把这个返回的函数看作一个计数器。makecounter 可以产生多个计数器，每个都独立计数。也就是说，
每个计数器函数都独享一个变量 t ，相互不干扰。这个 t 被称作计数器函数的 upvalue ，被绑定到计数器函
数中。拥有了 upvalue 的函数就是闭包。

- 函数和upvalue的类型定义被放在了`lobject.h`，这两个是lua的内置类型并不能被用户得到。
```
#define LUA_TPROTO	LUA_NUMTAGS		/* function prototypes */
#define LUA_TDEADKEY	(LUA_NUMTAGS+1)		/* removed keys in tables */
```

### 函数原型

- 首先来看下函数原型在lua里面的定义：
```
/*
** Function Prototypes
*/
// lobject.h
typedef struct Proto {
  CommonHeader;
  lu_byte numparams;  /* number of fixed parameters */
  lu_byte is_vararg;
  lu_byte maxstacksize;  /* number of registers needed by this function */
  int sizeupvalues;  /* size of 'upvalues' */
  int sizek;  /* size of 'k' */
  int sizecode;
  int sizelineinfo;
  int sizep;  /* size of 'p' */
  int sizelocvars;
  int linedefined;  /* debug information  */
  int lastlinedefined;  /* debug information  */
  TValue *k;  /* constants used by the function */
  Instruction *code;  /* opcodes */
  struct Proto **p;  /* functions defined inside the function */
  int *lineinfo;  /* map from opcodes to source lines (debug information) */
  LocVar *locvars;  /* information about local variables (debug information) */
  Upvaldesc *upvalues;  /* upvalue information */
  struct LClosure *cache;  /* last-created closure with this prototype */
  TString  *source;  /* used for debug information */
  GCObject *gclist;
} Proto;

```
- 从数据结构体中可以看出，里面包含了很多debug所需要的信息，包含了函数引用的常量表、调试信息。以及有多少个参数，调用这个函数需要多大的数据空间。
- lua 将原型和变量绑定的过程，都尽量避免重复生成不必要的闭包。当生成一次闭包
后，闭包将被`cache` 引用，下次再通过这个原型生成闭包时，比较 `upvalue` 是否一致来决定复用。`cache` 是
一个弱引用，一旦在 `gc` 流程发现引用的闭包已不存在，`cache` 将被置空。

### 闭包

- 接着看下闭包的数据结构：
```
// lobject.h
/*
** Closures
*/
// Lua支持的两种闭包，由Lua语言实现的，以及用C语言实现的。
// 他们都同属于lua定义的数据类型 LUA_TFUNCTION
#define ClosureHeader \
	CommonHeader; lu_byte nupvalues; GCObject *gclist

typedef struct CClosure {
  ClosureHeader;
  lua_CFunction f;
  TValue upvalue[1];  /* list of upvalues */
} CClosure;


typedef struct LClosure {
  ClosureHeader;
  struct Proto *p;
  UpVal *upvals[1];  /* list of upvalues */
} LClosure;


typedef union Closure {
  CClosure c;
  LClosure l;
} Closure;

```