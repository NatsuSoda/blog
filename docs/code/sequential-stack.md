---
title: 顺序栈：一段动态数组实现的栈，以及 8 个必须修的边界问题
description: 拆解 stack.cpp 这份顺序栈作业代码：base/top/stacksize 三个成员如何配合，push/pop/gettop 的指针动作，实测输出与扩容行为，以及实际运行中暴露的 8 个边界问题（含堆越界崩溃复现）和一份修正版实现。
date: 2026-10-22
tags:
  - 数据结构
  - 栈
  - C++
  - 代码笔记
---

# 顺序栈：一段动态数组实现的栈，以及 8 个必须修的边界问题

栈大概是数据结构里接口最短的一个：只看 `push` 和 `pop` 两个动作，LIFO 的规则一句话就能说完。但真正把它写成能用的代码时，问题几乎全藏在「指针指到哪」「满了怎么办」「空了怎么办」这三件事上。

这篇文章基于我写的一份课程作业 `stack.cpp`——用一段 `malloc` 出来的连续空间实现的顺序栈。代码只有 96 行，`main` 里压 10 个随机数再弹出来，运行结果完全正确。但我在它上面加了几组边界测试之后，发现了 8 个问题，其中一个会让程序在第 111 次 `push` 时悄悄写坏堆，直到 `free` 才崩溃。

下面先贴完整代码，再逐个操作拆开讲，最后给出实测证据和修正版。

> 实测环境：Windows + MinGW-w64 g++ 8.1.0，`-Wall -Wextra` 编译。文中所有输出都是真实运行结果，不是「应该会输出」。

## 一、完整代码

```cpp
#include <stdio.h>
#include <stdlib.h>
#define MAXSIZE 100
#define delta 10

typedef int elemtype;

typedef struct stacknode
{
	elemtype *top;
	elemtype *base;
	elemtype stacksize;
}stacknode;

int initstack(stacknode &s)
{
	s.base=(elemtype*)malloc(sizeof(elemtype)*MAXSIZE);
	if(!s.base)
	{
		printf("ERROR!");
	};
	s.top=s.base;
	s.stacksize=MAXSIZE;
	return 0; 
}

int destorystack(stacknode &s)
{
	if(!s.base){
		printf("ERROR!");
	}
	free(s.base);
	s.top=s.base=NULL;
	s.stacksize=0;
	return 0; 
}

int clearstack(stacknode &s){
	if(!s.base){
		printf("ERROR!");
	}
	s.top=s.base;
	return 0; 
}

int gettop(stacknode &s,elemtype e){
	if(s.base==s.top){
		printf("ERROR!");
	}
	e=*(s.top-1);
	printf("栈顶元素是：%d\n",e);
	return 0; 
}

int push(stacknode &s,elemtype e){
	if(s.top-s.base>=s.stacksize){
		s.base=(elemtype*)realloc(s.base,(s.stacksize+delta)*sizeof(elemtype));
		if(!s.base){
			printf("ERROR!");
		}
	}
	*(s.top++)=e;
	return 0; 
}

int pop(stacknode &s,elemtype &e){
	if(s.base==s.top){
		printf("ERROR!");
	}
	e=*(--s.top);
	return 0; 
}


int main(){
	stacknode sq;
	elemtype a;
	int i;
	if(!initstack(sq)){
		printf("OK!");
	}
	printf("Enter 10 elements:\n");
	for(i=1;i<=10;i++)
	{
		a=rand()%100;
		printf("%d\n",a);
		push(sq,a);}
	printf("Output 10 elements:\n");
	for(i=1;i<=10;i++){
		pop(sq,a);
		printf("%d\n",a);	
	}		
	return 0;
}
```

整体结构很典型：`init / destroy / clear / gettop / push / pop` 六个操作，加一个演示用的 `main`。下面按「数据结构 → 初始化 → 入栈 → 出栈 → 取栈顶 → 清空销毁」的顺序拆。

## 二、三个成员：`base`、`top`、`stacksize`

顺序栈的全部状态就靠结构体里这三个成员：

| 成员 | 含义 | 初始化后 |
|------|------|----------|
| `base` | 指向这段连续空间的首地址，一旦分配就不再移动（除非扩容） | `malloc` 返回的块首 |
| `top` | 指向**下一个可写入的空位**，不是当前栈顶元素 | 等于 `base` |
| `stacksize` | 当前这段空间最多能放多少个元素 | `MAXSIZE = 100` |

这里最容易混淆的是 `top` 的语义。它指向「栈顶元素的下一个位置」，所以：

- **空栈**：`top == base`
- **元素个数**：`top - base`，不需要额外的 `length` 计数器
- **栈满**：`top - base == stacksize`

```
空栈                              push(11) 之后               再 push(22)
base/top                         base   top                   base        top
  │                                │      │                     │           │
  ▼                                ▼      ▼                     ▼           ▼
┌────┬────┬────┬────┐           ┌────┬────┬────┬────┐        ┌────┬────┬────┬────┐
│    │    │    │    │           │ 11 │    │    │    │        │ 11 │ 22 │    │    │
└────┴────┴────┴────┘           └────┴────┴────┴────┘        └────┴────┴────┴────┘
 top-base = 0                    top-base = 1                 top-base = 2
```

用指针差代替长度计数器，是这份代码里设计得最漂亮的一处：所有判空、判满、求长度都围绕 `top - base` 展开，不会出现「`length` 和实际元素数不一致」这种经典 bug。但反过来，一旦**指针本身失效**（扩容搬移、释放后继续用），`top - base` 就变成了两个无关地址相减，错误会被彻底掩盖——这正是后面几个问题的根源。

另外注意成员声明顺序是 `top, base, stacksize`，但初始化顺序是 `base → top → stacksize`，读代码时别被结构体里的顺序带跑。

## 三、初始化：分配 100 个格子

```cpp
int initstack(stacknode &s)
{
	s.base=(elemtype*)malloc(sizeof(elemtype)*MAXSIZE);
	if(!s.base)
	{
		printf("ERROR!");
	};
	s.top=s.base;
	s.stacksize=MAXSIZE;
	return 0; 
}
```

`malloc` 出 100 个 `int`（400 字节），`base` 指向块首，`top` 也指向块首，容量记为 100。初始化后的状态实测：

```
after init               stacksize=100 base=00B21410 top=00B21410 top-base=0
```

`base == top` 就是空栈的判据。这一点和链表用 `L->next == NULL` 判空是同一个思路：**给「空」一个明确、可比较的表示**。

但这里有几个值得注意的地方：

**第一，分配失败后没有返回。** `printf("ERROR!")` 之后函数继续往下执行，照样把 `stacksize` 设成 100 并返回 `0`。此时 `base = top = NULL`，任何一次 `push` 都会在 `if(s.top-s.base>=s.stacksize)` 处得到 `0 >= 100`（假），于是直接执行 `*(s.top++) = e`——向地址 0 写入，程序当场崩溃。错误被打印了，却没有被处理。

**第二，返回值是 `0`，而 `main` 里的判断是 `if(!initstack(sq))`。** `0` 取反为真，所以成功时打印 `OK!`。这行代码恰好能跑对，但它表达的是「返回 0 表示成功」，而 `main` 的写法读起来更像「返回假表示成功」，语义是拧着的。教材里通常约定 `OK = 1`、`ERROR = 0`，调用方写 `if (initstack(sq) == OK)`；这份代码把两套习惯混在了一起。

**第三，`OK!` 后面没有换行**，紧接着的 `Enter 10 elements:` 会粘在同一行。实测的第一行输出就是：

```
OK!Enter 10 elements:
```

这种细节不影响功能，但会让人误以为程序输出格式坏了。

## 四、push：先判满，再扩容，最后写入

```cpp
int push(stacknode &s,elemtype e){
	if(s.top-s.base>=s.stacksize){
		s.base=(elemtype*)realloc(s.base,(s.stacksize+delta)*sizeof(elemtype));
		if(!s.base){
			printf("ERROR!");
		}
	}
	*(s.top++)=e;
	return 0; 
}
```

`push` 的动作可以拆成三步：

1. **判满**：`top - base >= stacksize`。相等就说明 100 个格子全用完了。
2. **扩容**：`realloc` 到 `stacksize + delta` 个元素，即每次多给 10 个格子。
3. **写入并移动栈顶**：`*(s.top++) = e`。后置 `++` 的含义是「先把 `e` 写到 `top` 当前指向的位置，再让 `top` 前移一格」，所以写完 `top` 恰好指向下一个空位，语义自洽。

`push` 与 `pop` 的指针动作是严格对称的：

```cpp
*(s.top++) = e;    // push：先写，后移
e = *(--s.top);    // pop ：先移，后读
```

一个用后置自增，一个用前置自减，方向相反、顺序也相反，记住这一对就不会写错。

`main` 里压入 10 个 `rand() % 100`，实测输出（`rand()` 未播种，序列固定）：

```
41
67
34
0
69
24
78
58
62
64
```

10 个元素远没到 100 的容量，所以这段演示**一次扩容都不会触发**。`realloc` 那条分支在 `main` 里是死代码——这正是边界问题能长期潜伏的原因。

## 五、pop：先移栈顶，再取值

```cpp
int pop(stacknode &s,elemtype &e){
	if(s.base==s.top){
		printf("ERROR!");
	}
	e=*(--s.top);
	return 0; 
}
```

`--s.top` 先把 `top` 退回到栈顶元素的位置，再解引用取出。被弹出的值通过引用参数 `elemtype &e` 带出函数，这是教材里标准的「返回值报状态、引用参数带数据」风格。

`main` 里连续弹 10 次，因为栈是后进先出，输出正好是入栈顺序的逆序：

```
Output 10 elements:
64
62
58
78
24
69
0
34
67
41
```

这段输出本身就是对 LIFO 最好的验证：最后压进去的 64 第一个出来，最早压进去的 41 最后出来。

和 `push` 对比一下引用参数的使用：`push` 的 `e` 是值传递（只需要读），`pop` 的 `e` 是引用传递（需要写回调用方）。这个区分是对的。问题出在同样需要「带出数据」的 `gettop` 上，下一节就能看到。

## 六、gettop：打印对了，但值没带出来

```cpp
int gettop(stacknode &s,elemtype e){
	if(s.base==s.top){
		printf("ERROR!");
	}
	e=*(s.top-1);
	printf("栈顶元素是：%d\n",e);
	return 0; 
}
```

`top` 指向下一个空位，所以栈顶元素在 `top - 1`，读法是对的。函数内部也确实打印出了正确的栈顶值。但注意参数列表：**`elemtype e` 是值传递，不是 `elemtype &e`**。

这意味着 `e = *(s.top - 1)` 只修改了函数内部的副本，调用方传进来的变量从头到尾没被碰过。实测：

```
== 3. gettop (e passed by value) ==
栈顶元素是：33
caller e after gettop = -1
```

进函数前 `e = -1`，函数里打印了 33，出来后 `e` 仍然是 `-1`。如果调用方写：

```cpp
elemtype top;
gettop(s, top);
use(top);        // 这里拿到的是未初始化的垃圾值
```

就会拿到一个完全没被赋值的变量。这个 bug 不会报错、不会崩溃，只会让结果莫名其妙——**比崩溃更难查**。

修正很简单，把参数改成引用：

```cpp
int gettop(stacknode &s, elemtype &e){
	if(s.base==s.top) return ERROR;   // 空栈也要返回
	e = *(s.top - 1);
	return OK;
}
```

顺带说一个设计问题：这个函数名叫 `gettop`，职责应该是「取栈顶」，但它内部直接 `printf`。取值和输出绑在一起，调用方就没法用这个值做别的事（比如比较、参与运算）。**取值函数只返回值，打印交给调用方**，是更清晰的职责划分。

## 七、clear 与 destroy：逻辑清空和物理释放

```cpp
int clearstack(stacknode &s){
	if(!s.base){
		printf("ERROR!");
	}
	s.top=s.base;
	return 0; 
}

int destorystack(stacknode &s)
{
	if(!s.base){
		printf("ERROR!");
	}
	free(s.base);
	s.top=s.base=NULL;
	s.stacksize=0;
	return 0; 
}
```

两个函数很容易被当成一回事，实际区别很大：

| 操作 | `top` | `base` | `stacksize` | 内存 |
|------|-------|--------|-------------|------|
| `clearstack` | 回到 `base` | 不变 | 100（保留） | 仍被占用，可继续用 |
| `destorystack` | `NULL` | `NULL` | 0 | 归还给系统 |

实测状态变化：

```
after clear              stacksize=100 base=00B21410 top=00B21410 top-base=0
after destroy            stacksize=0   base=00000000 top=00000000 top-base=0
```

`clearstack` 只是把 `top` 拨回起点，100 个格子的内存一个字节都没动，原来的数据还留在里面，只是逻辑上「不存在」了。`destorystack` 则真的 `free` 掉，并把三个成员全部归零。

有一个细节值得注意：`destorystack` 里 `if(!s.base)` 只打印错误，但后面仍然执行 `free(s.base)`。对 `NULL` 调用 `free` 是合法的（什么都不做），所以重复 `destroy` 不会崩溃，只会多打印一句 `ERROR!`。`clearstack` 对已销毁的栈调用也类似：打印错误后执行 `s.top = s.base`，把 `NULL` 赋给 `NULL`，同样安全。这两个函数的「失败路径」碰巧是安全的，但并不是因为代码写对了，而是因为标准库对 `free(NULL)` 的宽容。

## 八、main 与实测输出

```cpp
int main(){
	stacknode sq;
	elemtype a;
	int i;
	if(!initstack(sq)){
		printf("OK!");
	}
	printf("Enter 10 elements:\n");
	for(i=1;i<=10;i++)
	{
		a=rand()%100;
		printf("%d\n",a);
		push(sq,a);}
	printf("Output 10 elements:\n");
	for(i=1;i<=10;i++){
		pop(sq,a);
		printf("%d\n",a);	
	}		
	return 0;
}
```

完整运行结果：

```
OK!Enter 10 elements:
41
67
34
0
69
24
78
58
62
64
Output 10 elements:
64
62
58
78
24
69
0
34
67
41
```

这里有几个和「栈」本身无关、但很值得说的细节：

- **`printf("Enter 10 elements:")` 但没有任何输入语句**。程序并没有读入 10 个数，而是用 `rand()` 自己生成。提示语和实际行为不符，如果这是课程作业，答辩时很容易被问住。
- **`rand()` 没有 `srand()` 播种**，所以每次运行都是同一串 41、67、34……。这有利于复现输出，但严格说不是「随机」。
- **`main` 最后没有调用 `destorystack`**。进程退出时操作系统会回收内存，所以这里不会有实际后果；但作为一个完整的栈实现，使用完应该显式销毁，养成习惯后在长期运行的程序里才不会漏。
- **`return 0` 是对的**。`main` 返回 0 表示正常退出，这一点比不少课程示例里 `return 1` 的写法规范。

## 九、复杂度与选型

先看这份实现的操作代价：

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| `initstack` | O(1) | 一次 `malloc` |
| `push` | 最坏 O(n) | 扩容时要搬移全部元素；翻倍扩容均摊 O(1)，本代码的固定增量均摊 O(n) |
| `pop` | O(1) | 只移动指针 |
| `gettop` | O(1) | 一次解引用 |
| `clearstack` | O(1) | 只改 `top` |
| `destorystack` | O(1) | 一次 `free` |

顺序栈和顺序表一样，`push/pop` 都只动栈顶，所以是 O(1)；唯一的 O(n) 出现在扩容搬移时。

但「均摊 O(1)」这句话成立是有前提的：**容量必须按几何级数增长**（比如每次翻倍）。这份代码用的是固定增量 `delta = 10`，容量从 100 开始按 100、110、120……线性增长。假设总共压入 n 个元素，扩容次数约为 n/10，第 k 次扩容要搬移约 100 + 10k 个元素，总搬移量是：

```
(100 + 110 + 120 + ... + n) ≈ n² / 20
```

也就是说，**固定增量扩容的总代价是 O(n²)，均摊到每次 push 是 O(n)**，而不是 O(1)。压 10000 个元素时，翻倍扩容总共大约搬 2n 个元素（2 万次），固定增量要搬约 500 万次——差了 200 多倍。

实际工程里通常这么选：

| 方案 | 扩容策略 | 优点 | 缺点 |
|------|---------|------|------|
| 顺序栈（本代码） | `realloc` 连续空间 | 缓存友好、实现简单、`top/base` 指针直观 | 扩容要搬移；固定增量会退化成 O(n²) |
| 顺序栈（翻倍） | 容量 ×2 | 均摊 O(1)，最常用 | 仍可能一次搬移大量元素 |
| 链栈 | 每次 `malloc` 一个结点 | 不用连续空间，理论上没有「满」 | 每结点多一个指针，缓存不友好，`malloc` 有开销 |
| `std::stack` | 默认基于 `deque` | 现成、异常安全、接口完整 | 默认不连续，不能随机访问 |

栈的典型应用也能解释为什么 O(1) 的 `push/pop` 重要：括号匹配、后缀表达式求值、函数调用栈、DFS 的非递归写法、编辑器的撤销栈——这些都是「高频压弹、只在栈顶操作」的场景，顺序栈的连续内存优势会非常明显。

## 十、这份代码里必须修的 8 个问题

前面按正常路径讲完了。下面是我加边界测试后发现的 8 个问题，按严重程度排序。每条都给出实测证据，最后一节给修正版完整代码。

### 1. 扩容后没有更新 `stacksize`：第 111 次 push 写坏堆

这是最严重的一个。`realloc` 把空间扩大到了 `stacksize + delta` 个元素，但 `s.stacksize` 从头到尾没变：

```cpp
s.base=(elemtype*)realloc(s.base,(s.stacksize+delta)*sizeof(elemtype));
// 少了 s.stacksize += delta; 或 s.stacksize = newsize;
```

后果是：**从第 101 次 `push` 开始，每次都会进入扩容分支**，而且每次都按 `100 + 10 = 110` 个元素去 `realloc`，请求的大小恒为 440 字节。实测：

```
after push 100           stacksize=100 base=00B21410 top=00B215A0 top-base=100
after push               stacksize=100 base=00B21410 top=00B215A4 top-base=101
after push               stacksize=100 base=00B21410 top=00B215A8 top-base=102
...
after push               stacksize=100 base=00B21410 top=00B215B4 top-base=105
```

`top-base` 一路涨到 105，`stacksize` 却始终是 100。物理上这块内存已经被 `realloc` 扩到 110 个 `int`（440 字节），所以第 101～110 次 `push` 看起来「正常」——数据确实写进去了。但**第 111 次 `push` 时，`top-base = 110 >= 100` 成立，`realloc` 仍然只要 440 字节，不会真的变大，随后写入下标 110，越界 4 字节**。

我压了 115 个元素，前 110 个值都正确，程序也没有立刻报错；但最后调用 `destorystack` 时直接崩溃：

```
push 115: top-base=115 stacksize=100
base[108..114] = 109 110 111 112 113 114 115
[exit=-1073740940]
```

退出码 `-1073740940` 即 `0xC0000374`，Windows 的堆损坏错误——越界写把堆的元数据踩坏了，`free` 的时候才被检测出来。**这就是最典型的「越界写不一定当场崩，而是在下一次内存操作时爆炸」。**

还有一种更隐蔽的情况：这次运行 `realloc` 恰好原地扩展，`base` 没变，所以 `top` 仍然有效；一旦堆上有碎片、`realloc` 把块搬到新地址，`top` 还指向旧块，**从第 101 次 push 起就是 use-after-free**。也就是说，这段代码在扩容后的行为取决于堆的心情，不可预测。

修法只有一句，但必须加在 `realloc` 成功之后：

```cpp
s.stacksize = s.stacksize + delta;
```

### 2. `realloc` 的返回值直接赋给 `base`，失败就丢指针

```cpp
s.base=(elemtype*)realloc(s.base,(s.stacksize+delta)*sizeof(elemtype));
if(!s.base){
	printf("ERROR!");
}
```

这是 `realloc` 的经典错误用法：

- **失败时**：`realloc` 返回 `NULL`，但原来的块**并没有被释放**。直接写回 `s.base` 就把唯一的指针弄丢了，原块永久泄漏；随后代码只打印一句 `ERROR!`，继续执行 `*(s.top++) = e`，而此时 `base = NULL`，栈状态彻底损坏。
- **成功但搬移时**：`s.top` 没有被同步调整。`top` 仍然指向旧地址，`base` 已经指向新地址，`top - base` 变成两个无关地址相减，之后再 `push/pop` 全是未定义行为。

正确写法是先接住返回值，再根据情况更新：

```cpp
elemtype *newbase = (elemtype*)realloc(s.base, (s.stacksize + delta) * sizeof(elemtype));
if (!newbase) return ERROR;          // 原块仍然有效，base 不动
s.top = newbase + (s.top - s.base);  // 用旧 base 算偏移，再重定位 top
s.base = newbase;
s.stacksize += delta;
```

注意 `s.top = newbase + (s.top - s.base);` 这一句的求值顺序：赋值号右边先算，此时 `s.top` 和 `s.base` 都还是旧值，指针差是合法的；算完再写 `s.top`，最后才更新 `s.base`。如果先写 `s.base = newbase`，`s.top - s.base` 就变成新旧指针混算，偏移量就错了。

### 3. `gettop` 的 `e` 是值传递，调用方拿不到栈顶值

前面第六节已经实测过：

```
栈顶元素是：33
caller e after gettop = -1
```

函数内部打印 33，调用方变量仍是 -1。改成 `elemtype &e` 即可；同时应该去掉函数内的 `printf`，让 `gettop` 只负责取值。

### 4. 空栈时 `pop` / `gettop` 打印错误后继续执行

两个函数都是这个模式：

```cpp
if(s.base==s.top){
	printf("ERROR!");
}
e=*(--s.top);       // 空栈时 top == base，--top 跑到 base 之前
```

打印 `ERROR!` 之后**没有 `return`**，继续往下走。空栈时：

- `pop`：`--s.top` 让 `top` 退到 `base - 1`，然后读取 `base` 之前的内存。实测弹出的值是一段垃圾：

```
ERROR!pop on empty -> e = 268448664
after pop on empty       stacksize=100 base=00B21410 top=00B2140C top-base=-1
```

`top-base = -1`，栈顶指针已经跑到缓冲区外面了。如果此时再 `push`，`*(s.top++)` 会写到 `base - 1`，**越界写发生在缓冲区之前**，破坏的是堆块头部。

- `gettop`：空栈时读 `*(top - 1)`，同样读到 `base` 之前的内存。实测打印了一个未初始化的值 `268452608`，而调用方什么都没得到。

修法是在判断后直接返回错误：

```cpp
if (!s.base || s.top == s.base) return ERROR;
```

`!s.base` 这个条件不能省：`destroy` 之后 `base` 和 `top` 都是 `NULL`，`top == base` 成立，但此时栈已经不存在了，应当返回错误而不是继续操作。

### 5. `destroy` 之后再 `push`：空指针解引用直接崩溃

`destorystack` 把 `base`、`top` 置为 `NULL`、`stacksize` 置 0。此时调用 `push`：

- 判满条件 `top - base >= stacksize` 在这里算出来是 `0 >= 0`，**成立**（严格说两个空指针相减也是未定义行为，只是实测恰好得到 0）；
- 进入扩容分支，`realloc(NULL, (0+10)*4)` 成功返回一块新内存，赋给 `base`；
- 但 `top` 仍然是 `NULL`；
- 执行 `*(s.top++) = e`，向地址 0 写入。

实测：

```
after destroy: base=0000000000000000 top=0000000000000000 stacksize=0
now push 1 ...
[exit=-1073741819]
```

退出码 `-1073741819` 即 `0xC0000005`，访问冲突。`push` 必须先检查 `s.base` 是否存在，不存在就返回错误，而不是「顺手 realloc 一下继续用」。销毁后的栈应该视为不可用，重新使用前必须重新 `init`。

### 6. `initstack` 永远返回 0，`main` 的判断语义是反的

`initstack` 无论 `malloc` 成功还是失败，最后都执行 `return 0`。而 `main` 里写的是：

```cpp
if(!initstack(sq)){
	printf("OK!");
}
```

`0` 取反为真，所以成功时打印 `OK!`——碰巧对了。但这段代码同时意味着：**失败时也返回 0，也打印 `OK!`**。初始化失败被完全掩盖，后续 `push` 会写空指针崩溃。

正确的做法是让返回值有区分度，调用方按约定判断：

```cpp
#define OK 1
#define ERROR 0

int initstack(stacknode &s)
{
	s.base = (elemtype*)malloc(sizeof(elemtype) * MAXSIZE);
	if (!s.base) return ERROR;
	s.top = s.base;
	s.stacksize = MAXSIZE;
	return OK;
}

/* 调用方 */
if (initstack(sq) != OK) {
	printf("init failed\n");
	return 1;
}
```

配套地，`push/pop/gettop/destroy` 也应该在失败时返回 `ERROR`，而不是一律 `return 0`。现在所有函数都返回 0，调用方根本没法区分成功和失败，`main` 里只检查了 `initstack`，其余调用的返回值全部被忽略。

### 7. 重复初始化会泄漏，失败路径也没有退出

```cpp
initstack(sq);   // 第一次 malloc，base 指向 400 字节
initstack(sq);   // 第二次 malloc，base 被覆盖
```

第二次调用没有检查 `base` 是否已经指向一块内存，直接覆盖指针，第一次分配的 400 字节就没人能释放了。虽然这个程序只初始化一次，但作为「基本操作」的实现，应该在文档里写明「每个栈只初始化一次」，或者加保护：

```cpp
if (s.base) return ERROR;   // 已初始化
```

如果要做这个检查，`stacknode` 变量必须先零初始化（`stacknode s = {NULL, NULL, 0};`），否则 `s.base` 是栈上的垃圾值，检查本身就会误判。这两个约定要配套使用。

同样的「失败继续执行」问题还出现在 `initstack` 的 `malloc` 失败分支和 `destorystack` 的空指针检查上：错误只打印、不返回，调用方看不出任何异常。

### 8. 命名、类型和提示语上的小问题

这些不影响运行，但放在一起会显得代码「没收拾过」：

| 位置 | 问题 | 建议 |
|------|------|------|
| `destorystack` | `destroy` 拼写错误 | 改为 `destroystack` / `DestroyStack` |
| `stacksize` | 类型用了 `elemtype`（即 `int`），容量类型和元素类型绑在一起 | 独立声明为 `int` / `size_t`；否则把 `elemtype` 改成 `char` 时容量会溢出 |
| `delta` | 宏名小写，和 `MAXSIZE` 风格不一致 | 统一为 `DELTA` / `GROW_STEP` |
| 函数命名 | 全小写、无统一前缀；`gettop` 与 `initstack` 风格相同但语义不同（一个打印、一个返回值） | 统一成 `InitStack` / `Push` / `Pop` / `GetTop` 或沿用教材的 `_Sq` 后缀 |
| `Enter 10 elements:` | 提示输入，但从未读取输入 | 要么改成 `Random 10 elements:`，要么真的用 `scanf` 读入 |
| `main` 结尾 | 没有 `destorystack(sq)` | 补上，养成「谁分配谁释放」的习惯 |
| `OK!` | 缺少换行 | 补 `\n`，避免和下一行输出粘连 |

单看每一条都是小事，但这份代码的问题恰恰集中在「边界」和「一致性」上：扩容的边界、空栈的边界、初始化的边界、命名的边界。**能跑的代码和能用的代码，差别就在这些地方。**

## 十一、修正版完整代码

把上面的问题都修掉之后，代码会长这样。接口保持课程作业的风格（`status` + `OK/ERROR`），但每个失败路径都有返回值，`push` 的扩容逻辑也补完整了：

```cpp
#include <stdio.h>
#include <stdlib.h>

#define INIT_SIZE 100
#define DELTA 10
#define OK 1
#define ERROR 0

typedef int elemtype;
typedef int status;

typedef struct stacknode
{
	elemtype *base;
	elemtype *top;
	int stacksize;
} stacknode;

status initstack(stacknode &s)
{
	if (s.base) return ERROR;          // 已经初始化过，避免覆盖旧指针造成泄漏
	s.base = (elemtype *)malloc(sizeof(elemtype) * INIT_SIZE);
	if (!s.base) return ERROR;
	s.top = s.base;
	s.stacksize = INIT_SIZE;
	return OK;
}

status destorystack(stacknode &s)
{
	if (!s.base) return ERROR;
	free(s.base);
	s.base = s.top = NULL;
	s.stacksize = 0;
	return OK;
}

status clearstack(stacknode &s)
{
	if (!s.base) return ERROR;
	s.top = s.base;
	return OK;
}

status gettop(stacknode &s, elemtype &e)
{
	if (!s.base || s.top == s.base) return ERROR;
	e = *(s.top - 1);
	return OK;
}

status push(stacknode &s, elemtype e)
{
	if (!s.base) return ERROR;
	if (s.top - s.base >= s.stacksize)
	{
		int newsize = s.stacksize + DELTA;
		elemtype *newbase = (elemtype *)realloc(s.base, sizeof(elemtype) * newsize);
		if (!newbase) return ERROR;                 // 原块仍然有效
		s.top = newbase + (s.top - s.base);         // 先按旧 base 算偏移
		s.base = newbase;
		s.stacksize = newsize;                      // 关键：更新容量
	}
	*(s.top++) = e;
	return OK;
}

status pop(stacknode &s, elemtype &e)
{
	if (!s.base || s.top == s.base) return ERROR;
	e = *(--s.top);
	return OK;
}

int main()
{
	stacknode s = {NULL, NULL, 0};
	elemtype a;
	int i;

	if (initstack(s) != OK)
	{
		printf("init failed\n");
		return 1;
	}

	for (i = 0; i < 10; i++)
	{
		a = rand() % 100;
		push(s, a);
	}
	printf("after 10 push: capacity=%d size=%d\n", s.stacksize, (int)(s.top - s.base));

	for (i = 0; i < 10; i++)
	{
		if (pop(s, a) == OK) printf("%d\n", a);
	}
	printf("after 10 pop : capacity=%d size=%d\n", s.stacksize, (int)(s.top - s.base));

	for (i = 0; i < 105; i++) push(s, i);
	printf("after 105 push: capacity=%d size=%d\n", s.stacksize, (int)(s.top - s.base));

	destorystack(s);
	printf("after destroy: base=%p top=%p capacity=%d\n",
	       (void *)s.base, (void *)s.top, s.stacksize);
	return 0;
}
```

实测输出：

```
after 10 push: capacity=100 size=10
64
62
58
78
24
69
0
34
67
41
after 10 pop : capacity=100 size=0
after 105 push: capacity=110 size=105
after destroy: base=0000000000000000 top=0000000000000000 capacity=0
```

重点看 `after 105 push` 那一行：压入 105 个元素后，容量从 100 涨到了 **110**，而不再是原版那个永远停在 100 的 `stacksize`。整个过程中只发生了一次扩容（100 → 110），因为 105 还没超过 110；如果继续压到 111，才会触发下一次 110 → 120。

修正版里唯一没有改变的是扩容策略：仍然是每次 `+10`。对课程作业来说这足够了；如果要处理大量数据，把 `DELTA` 改成「容量翻倍」就够了：

```cpp
int newsize = s.stacksize * 2;
```

## 十二、结语

这份 96 行的代码，正常路径上几乎无可挑剔：`top` 指向下一个空位的约定、`top - base` 求元素个数、`push` 的 `*top++` 与 `pop` 的 `*--top` 对称、`clear` 与 `destroy` 的职责区分，都是标准写法。`main` 的输出也完全正确。

但它的问题全在正常路径之外：容量扩大后忘了记账、`realloc` 失败后继续写、空栈时打印错误却不返回、销毁后还能被 `push` 访问。**这些分支在「压 10 个再弹 10 个」的演示里一次都不会执行**，所以代码看起来毫无问题，直到输入变成 111 个、或者用户在空栈上多点了一次。

我自己的收获是两点：

第一，**指针实现的容器，最脆弱的不是元素，而是指针本身**。`top - base` 这种写法很优雅，但它把「长度」寄托在两个指针的有效性上。扩容、释放、搬移，任何一次指针失效都会让长度信息一起失真，而且不会报错。

第二，**测试用例要专门往边界上撞**。压 10 个元素测不出扩容 bug，压 100 个测不出，压 111 个才会写坏堆；空栈 `pop` 一次不崩，崩溃发生在下一次 `push`。写完一个容器，至少应该测：空、满、刚超容量、扩容后、销毁后再操作这五种状态。这比对着代码看十遍有用得多。

顺带一提：这几种状态我在验证时是用一份单独的测试程序跑的（把原文件 `#include` 进来、用宏把原 `main` 改名，再写自己的 `main`），这样既不用改动作业源码，又能精确控制压栈次数。这个方法对整理其他课程代码同样适用。
