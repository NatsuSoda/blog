---
title: 顺序表与链表：两份 C/C++ 代码的逐行整理与实现对比
description: 把课程作业里的「静态数组顺序表」与「单链表」两份代码逐行拆开整理：结构定义、初始化、查找、插入、删除全部过一遍，附实际运行结果、复杂度推导、边界行为的实测验证，以及两个文件里值得注意的 7 个细节。
date: 2026-10-16
tags:
  - 数据结构
  - C语言
  - 线性表
  - 代码笔记
---

# 顺序表与链表：两份 C/C++ 代码的逐行整理与实现对比

线性表是数据结构课的第一站，也是「看得懂 ≠ 写得出来」这句话第一次生效的地方。课本上的伪代码往往用 `elem[i]`、`p->next` 一笔带过，真正落到能编译运行的 `.cpp` 文件里，边界判断、下标偏移、循环方向全是坑。

这篇文章把我写的两份课程代码完整整理一遍：

| 文件 | 实现 | 存储方式 |
|------|------|----------|
| `2.1实现顺序表（静态数组形式）的基本操作.cpp` | 顺序表 `SqList` | 一段连续内存 + 长度 / 容量两个计数器 |
| `Linklist.cpp` | 单链表 `Linklist` | 带头结点的链式存储 |

两份代码的运行结果、边界行为我都实际编译跑过一遍，文中给出的输出是真实结果，不是「应该会输出」。下面先给完整代码，再逐个操作拆解。

## 一、先看全貌：两份代码的完整实现

### 1. 顺序表（静态数组形式）

```c
#include <stdio.h>
#include <stdlib.h>
#define OK 1
#define ERROR 0
#define OVERFLOWE 2
#define INIT_LIST_SIZE 100

typedef int Status;
typedef int ElemType;
typedef struct
{
	ElemType * elem;
	int length;
	int listsize;
}SqList;

Status InitList_Sq(SqList &L)
{
	L.elem = (ElemType *)malloc(sizeof(ElemType)*INIT_LIST_SIZE);
	if(!L.elem) return OVERFLOWE;
	L.length = 0;
	L.listsize = INIT_LIST_SIZE;
	return OK;
}

Status DestroyList_Sq(SqList &L)
{
	if(!L.elem)return ERROR;
	free(L.elem);
	L.length = 0;
	L.listsize = 0;
	L.elem = NULL;
	return OK;
}

Status ClearList_Sq(SqList &L)
{
	if(!L.elem)return ERROR;
	L.length = 0;
	return OK;
}

Status GetElemList_Sq(SqList &L, int i, ElemType &e)
{
	if(i<1||i>L.length) return ERROR;
	e = L.elem[i-1];
	return OK;
}

Status InsertList_Sq(SqList &L, int i, ElemType e)
{
	int j;
	if(i<1||i>L.length+1) return ERROR;
	if(L.length>=L.listsize) return ERROR;
	for(j=L.length - 1;j>=i-1; j--)
	{
		L.elem[j+1] = L.elem[j];
	}
	L.elem[i-1] = e;
	L.length++;
	return OK;
}

Status DeleteList_Sq(SqList &L, int i, ElemType &e)
{
	int j;
	if(i<1||i>L.length) return ERROR;
	e = L.elem[i-1];
	for(j=i;j<L.length; j++)
	{
		L.elem[j-1] = L.elem[j];
	}
	L.length--;
	return OK;
}

int main(){
	int i;
	ElemType e;
	SqList list;
	InitList_Sq(list);
	for(i=1; i<=10; i++){
		e = rand()%100;
		InsertList_Sq(list, i, e);
	}
	for(i=1; i<=10; i++){
		GetElemList_Sq(list, i, e);
		printf("%d\n", e);
	}
	for(i=1; i<=10; i++){
		DeleteList_Sq(list, 1, e);
		printf("%d is deleted\n", e);
	}
	printf("List length:%d\n", list.length);
	return 1;
}
```

### 2. 单链表（带头结点）

```c
#include<stdio.h>
#include<stdlib.h>
#define OK 1
#define ERROR 0
#define OVERFLOW -1
#define INITLISTSIZE 100


typedef int Status;
typedef int ElemType;


typedef struct Lnode
{
	ElemType data;
	struct Lnode *next;

}Lnode,*Linklist;


Status INIT_List(Linklist &L)
{

L=(Linklist)malloc(INITLISTSIZE*sizeof(Lnode));

if(!L) 
{
return ERROR;
}

L->next=NULL;
return OK;
}

Status GetElem(Linklist L,int i,ElemType &e)
{
	Linklist p;
	int j;
	p=L->next;
	j=1;
	while(p&&j<i)
	{
	p=p->next;j++;	
	}

if(!p||j>i) 
{
return ERROR;
}

e=p->data;

return OK;
}

Status Delete_List(Linklist &L,int i,ElemType &e)
{
	Linklist p,q;
	int j=0;
	p=L;

	while(p->next&&j<i-1)
	{
	p=p->next;
	j++;	
	}

	if(!p->next||j>i-1)
	{
		return ERROR;
	}
	q=p->next;

	e=q->data;

	p->next=q->next;
	free(q);
	return OK;
}

Status Insert_List(Linklist &L,int i,ElemType e)
{
	Linklist p,q;
	p=L;
	int j=0;
	while(p&&j<i-1)
	{
	p=p->next;j++;	
	}

	if(!p||j>i-1)
	{
		return ERROR;
	}
	q=(Linklist)malloc(sizeof(Lnode));
	q->data=e;
	q->next=p->next;
	p->next=q;
	return OK;
}

int main(){
	Linklist list;
	ElemType a;
	int i;
	if(!INIT_List(list)){
		exit(OVERFLOW);
	}
	printf("Insert 10 elements:\n");
	for(i=1;i<=10;i++){
		a=rand()%100;
		printf("%d\n",a);
		Insert_List(list,i,a);
	}
	printf("\nDelete 10 elements:\n");
	for(i=1;i<=10;i++)
	{Delete_List(list,1,a);
	printf("%d\n",a);	
	}
	return OK;
}
```

两份代码的结构完全对称：都是 `Init / Get / Insert / Delete` 四件套，都用 `Status` 返回 `OK` 或 `ERROR`，都在 `main` 里插 10 个随机数再删干净。**差异只在「怎么找到第 i 个位置」**——顺序表靠下标算地址，链表靠指针一路 `next` 走过去。

## 二、顺序表：下标算地址，代价是搬数据

### 结构定义与初始化

```c
typedef struct
{
	ElemType * elem;   // 指向动态分配的连续空间
	int length;        // 当前元素个数
	int listsize;      // 当前容量上限
}SqList;
```

这里有个容易被名字骗到的地方：文件名叫「静态数组形式」，但 `elem` 是指针，空间来自 `malloc`，所以**本质仍是动态数组**。所谓「静态」是指容量固定为 `INIT_LIST_SIZE = 100`，插满了不做扩容：

```c
Status InitList_Sq(SqList &L)
{
	L.elem = (ElemType *)malloc(sizeof(ElemType)*INIT_LIST_SIZE);
	if(!L.elem) return OVERFLOWE;
	L.length = 0;
	L.listsize = INIT_LIST_SIZE;
	return OK;
}
```

初始化后的状态是「有 100 个格子的空表」，用实测数据说就是：

```
空表 length=0, listsize=100
```

`length` 和 `listsize` 的区分是顺序表的核心：`length` 决定**逻辑上哪些格子有效**，`listsize` 决定**物理上最多能放多少**。所有越界判断都围绕 `length` 展开，而扩容判断看的是 `listsize`。

`DestroyList_Sq` / `ClearList_Sq` 的差别也要分清：

- `ClearList_Sq` 只把 `length` 归零，内存留着继续用——**逻辑清空**
- `DestroyList_Sq` 把内存 `free` 掉并置 `L.elem = NULL`——**物理销毁**，之后再操作会立刻返回 `ERROR` 而不是野指针崩溃

### 取值：O(1) 的随机访问

```c
Status GetElemList_Sq(SqList &L, int i, ElemType &e)
{
	if(i<1||i>L.length) return ERROR;
	e = L.elem[i-1];
	return OK;
}
```

**注释里最该记住的一句是 `i-1`**：教材的位置 `i` 从 1 开始数，C 数组下标从 0 开始，两者差 1。这个偏移在顺序表里出现三次（取值、插入、删除），写错就是经典的「差一位」bug。

复杂度是 **O(1)**，因为第 i 个元素的地址可以直接算出来：

```
&L.elem[i-1] = L.elem + (i-1) * sizeof(ElemType)
```

一次乘法加一次加法，跟表长无关——这就是「随机访问」，也是链表永远追不上的地方。

### 插入：从后往前挪

```c
Status InsertList_Sq(SqList &L, int i, ElemType e)
{
	int j;
	if(i<1||i>L.length+1) return ERROR;   // 合法位置：1 ~ length+1
	if(L.length>=L.listsize) return ERROR; // 表满则不插
	for(j=L.length - 1;j>=i-1; j--)
	{
		L.elem[j+1] = L.elem[j];
	}
	L.elem[i-1] = e;
	L.length++;
	return OK;
}
```

两个上界要区分开：

- **插入**的合法范围是 `1 ~ length+1`，因为可以在表尾追加（`i = length+1`）
- **删除**和**取值**的范围只有 `1 ~ length`，因为不能删一个不存在的元素

循环方向必须是**从后往前**。假设表里是 `[1,2,3,4,5]`，要在第 3 位插入 `99`：

| 步骤 | 操作 | 结果 |
|------|------|------|
| 初始 | — | 1 2 3 4 5 |
| j=4 | `elem[5] = elem[4]` | 1 2 3 4 5 **5** |
| j=3 | `elem[4] = elem[3]` | 1 2 3 4 **4** 5 |
| j=2 | `elem[3] = elem[2]` | 1 2 3 **3** 4 5 |
| 收尾 | `elem[2] = 99` | 1 2 **99** 3 4 5 |

如果反过来从前往后挪，`elem[3] = elem[2]` 先把 3 覆盖成 99 的位置，下一步再挪就会把刚写进去的值冲掉，结果是一串重复元素。**这个循环方向是考试和面试的高频追问点。**

时间复杂度：最好情况在表尾插入（`i = length+1`），循环一次都不执行，O(1)；最坏在表头插入，移动全部 n 个元素，O(n)；平均移动 n/2 个，仍是 **O(n)**。

表满时的行为我实测过：

```
塞满 100 个后 length=100
表满再 InsertList_Sq(第101位) 返回=0  length=100
表满再 InsertList_Sq(第102位) 返回=0  length=100
```

返回 `0`（即 `ERROR`），`length` 保持 100 不变，数据没被破坏——**但元素也没插进去**。这是「静态容量」的硬伤：到了 100 就再也放不下了。真正工程化的做法是扩容，也就是把这里改成 realloc 出一块 2 倍空间再拷过去，这就是「动态数组」的由来。

### 删除：从前往后挪

```c
Status DeleteList_Sq(SqList &L, int i, ElemType &e)
{
	int j;
	if(i<1||i>L.length) return ERROR;
	e = L.elem[i-1];          // 先用 e 带出被删的值
	for(j=i;j<L.length; j++)
	{
		L.elem[j-1] = L.elem[j];
	}
	L.length--;
	return OK;
}
```

被删的元素通过引用参数 `ElemType &e` 带出去，这是教材里统一的「输出参数」风格：用返回值报成功失败，用引用参数带数据。

与插入正好相反，删除的搬移是**从前往后**，把后面的元素依次往前覆盖：

```
初始： 1 2 3 4 5   （删第 2 位）
j=2:  elem[1] = elem[2]  →  1 3 3 4 5
j=3:  elem[2] = elem[3]  →  1 3 4 4 5
j=4:  elem[3] = elem[4]  →  1 3 4 5 5
length-- 后有效区： 1 3 4 5      （末尾的 5 变成无效残留）
```

注意最后那个「残留的 5」——它还在内存里，只是超出了 `length` 的范围，逻辑上不存在了。这也是顺序表删除并不真的「清掉」数据的体现。平均同样要移动 n/2 个元素，**O(n)**。

### main：插 10 个、读 10 个、删 10 个

```c
for(i=1; i<=10; i++){
	e = rand()%100;
	InsertList_Sq(list, i, e);   // i 递增 → 依次尾插，得到原顺序
}
```

`i` 从 1 递增到 10，每次都插在当前表尾，所以最终顺序就是生成顺序。实测运行结果：

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
41 is deleted
...
64 is deleted
List length:0
```

删除循环是关键的一个技巧：

```c
for(i=1; i<=10; i++){
	DeleteList_Sq(list, 1, e);   // 每次都删「第 1 个」
	printf("%d is deleted\n", e);
}
```

每次都删第 1 位，所以输出顺序与插入顺序一致，删 10 次后 `length` 回到 0。如果这里写的是 `DeleteList_Sq(list, i, e)`，输出就会是原始顺序的错位结果——**想按顺序清空一张表，就反复删第一个位置。**

## 三、单链表：指针找位置，代价是走路

### 结构定义与头结点

```c
typedef struct Lnode
{
	ElemType data;
	struct Lnode *next;
}Lnode,*Linklist;
```

这里一行干了两件事：`Lnode` 是结点类型，`Linklist` 是指向结点的指针类型。链表操作里满屏的 `Linklist p, q` 其实都是结点指针。

初始化：

```c
Status INIT_List(Linklist &L)
{
	L=(Linklist)malloc(INITLISTSIZE*sizeof(Lnode));
	if(!L) return ERROR;
	L->next=NULL;
	return OK;
}
```

这个头结点就是**带头结点链表**的标志。它不存数据，只负责让「空表」和「第一个数据结点」有统一的处理方式。实测它的 `data` 是一段未初始化的垃圾值：

```
初始 H->data=12321104, H->next=0000000000000000
```

`H->next == NULL` 才是判空依据，`H->data` 从头到尾都不参与任何逻辑——**头结点的 data 域永远是废的，别去读它。**

带头结点最大的好处是**插入和删除不需要特判空表或表头**：因为没有头结点时，在第 1 位插入要修改头指针本身，得写成 `Linklist &L` 或者返回新头指针；有了头结点，第 1 位只是「头结点之后插入」，和中间位置走同一套逻辑。

### 取值：走 i-1 步

```c
Status GetElem(Linklist L,int i,ElemType &e)
{
	Linklist p;
	int j;
	p=L->next;      // p 从第 1 个数据结点开始
	j=1;
	while(p&&j<i)   // 走 i-1 步
	{
		p=p->next;j++;
	}
	if(!p||j>i) return ERROR;
	e=p->data;
	return OK;
}
```

链表没有「第 i 个元素的地址」这种算法，只能从头一步步走。循环条件 `p && j < i` 同时干了三件事：

- `p` 保证不会越过表尾去解引用空指针
- `j < i` 控制步数
- 循环结束后 `p` 若非空，恰好指向第 i 个结点

复杂度 **O(n)**——这正是链表相对顺序表丢掉的那一分：随机访问从 O(1) 退化成了 O(n)。

实测越界行为：

```
GetElem(0) r=0 e=3
GetElem(20) r=0 e=3
GetElem(1) r=1 e=1
```

越界时返回 `ERROR`，但 `e` 保持进入函数前的旧值（这里是 3）**没有被修改**。这是引用传参的一个隐蔽点：调用方如果不检查返回值，会拿着上一次的残留值继续算，而且完全不报错。

### 插入：先连后断，顺序不能反

```c
Status Insert_List(Linklist &L,int i,ElemType e)
{
	Linklist p,q;
	p=L;
	int j=0;
	while(p&&j<i-1)     // 找第 i-1 个结点
	{
		p=p->next;j++;
	}
	if(!p||j>i-1) return ERROR;
	q=(Linklist)malloc(sizeof(Lnode));
	q->data=e;
	q->next=p->next;    // ① 新结点先接上后继
	p->next=q;          // ② 前驱再指向新结点
	return OK;
}
```

**指针操作的顺序是这一节的全部重点。** 必须先把 `q->next` 指向 `p->next`，再改 `p->next`：

```
① q->next = p->next       ② p->next = q
   ┌────┐   ┌────┐            ┌────┐   ┌────┐   ┌────┐
   │ p  │──▶│ q  │──▶ ...     │ p  │──▶│ q  │──▶ ...
   └────┘   └────┘            └────┘   └────┘
      └──────────┘
```

如果两句写反，先执行 `p->next = q`，`p` 原本的后继地址就丢了，新结点的 `next` 再也接不上原来的链表——从 `q` 往后整段内存直接泄漏并丢失。**「先接后断」四个字要形成肌肉记忆。**

另外注意 `p = L`（而不是 `L->next`）与 `j = 0` 的配合：要找第 `i-1` 个结点，从头结点出发走 `i-1` 步；`i=1` 时循环不执行，`p` 停在头结点，新结点插在头结点之后，也就是「第 1 个位置」——**这是带头结点方案不特判表头的原因**。

两种建表方式在实测里表现得很清楚：

```
-- 连续 Insert_List(1,..) 10 次 --   逆序建表
  insert(1,10) r=1 -> 表: 10 | end
  insert(1,20) r=1 -> 表: 20 10 | end
  insert(1,30) r=1 -> 表: 30 20 10 | end
  ...
-- 顺序尾插 Insert_List(i,..) --     顺序建表
  insert(1,1) r=1
  insert(2,2) r=1
  ...
顺序建表: 1 2 3 4 5 | end
```

- 固定 `i=1` 反复插 → **头插法**，结果与输入顺序**相反**
- `i` 递增插 → **尾插法**，结果与输入顺序一致

原 `main` 用的是 `Insert_List(list,i,a)`，所以输出顺序和生成顺序相同。

越界插入也实测了，处理得很干净：

```
insert(4,99) r=1 -> 表: 1 2 3 99 4 5 | end
insert(20,77) r=0 -> 表: 1 2 3 99 4 5 | end
```

在 6 个元素中插第 20 位，返回 `ERROR`，链表保持原样没有破坏。

### 删除：记住前驱，才能摘链

```c
Status Delete_List(Linklist &L,int i,ElemType &e)
{
	Linklist p,q;
	int j=0;
	p=L;
	while(p->next&&j<i-1)   // 找第 i-1 个结点
	{
		p=p->next;
		j++;
	}
	if(!p->next||j>i-1) return ERROR;
	q=p->next;              // q 指向要删的结点
	e=q->data;              // 数据带出去
	p->next=q->next;        // 前驱跳过 q
	free(q);                // 释放 q
	return OK;
}
```

单链表删除必须**先找到前驱**，因为「摘链」这个动作只能由前驱完成：`p->next = q->next`。这也是为什么循环条件和插入一样是在找第 `i-1` 个结点。

同样有四步顺序，一步都不能少：

1. `q = p->next` —— 让 `q` 记住要删的结点（否则摘链后再也找不到它，`free` 无从下手）
2. `e = q->data` —— **先把数据带出去，再释放结点**，顺序反了就是 use-after-free
3. `p->next = q->next` —— 前驱绕过 `q`
4. `free(q)` —— 归还内存

对比顺序表：顺序表删除是「覆盖式」的，被删元素的内存还在，只是移出逻辑范围；链表删除是真正的**摘链 + free**，堆上那块内存被交还给系统。代价是每次都要从头找前驱，复杂度仍然是 **O(n)**。

在 `main` 里，清空链表用的是同一个技巧：

```c
for(i=1;i<=10;i++)
{
	Delete_List(list,1,a);   // 反复删第 1 个
	printf("%d\n",a);
}
```

每次删第 1 位，此时 `p = L` 已在前驱位置，循环 `while(p->next && j < i-1)` 中 `i-1 = 0`，一次都不执行，**定位是 O(1)**，整表清空是 O(n) 而不是 O(n²)。实测输出与插入顺序完全一致：

```
Delete 10 elements:
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

至于边界情况：

```
Delete_List(20) r=0 -> 表: 1 2 3 99 4 5 | end
空表 Delete_List(1) r=0
```

越界和空表都返回 `ERROR`，链表完好无损——`p->next` 这个判空条件在空表场景下刚好挡住了对空指针的解引用。

## 四、对比总结

| 对比项 | 顺序表 `SqList` | 单链表 `Linklist` |
|--------|----------------|-------------------|
| 存储方式 | 连续内存（`malloc` 一段） | 离散结点 + 指针串联 |
| 容量 | 固定 100，满则 `ERROR` | 用多少 `malloc` 多少 |
| 取值 `Get` | **O(1)**，下标直接算 | **O(n)**，需走 i-1 步 |
| 插入 | O(n)，平均移动 n/2 个 | O(n) 定位 + O(1) 接链 |
| 删除 | O(n)，平均移动 n/2 个 | O(n) 定位 + O(1) 摘链 |
| 已知位置时插删 | 仍需移动，O(n) | **O(1)**（这才是链表的主场） |
| 额外空间 | 每元素只有数据本身 | 每结点多一个 `next` 指针 |
| 缓存友好性 | 高（连续访问） | 低（指针跳转） |
| 判空 | `length == 0` | `L->next == NULL` |
| 容量越界 | 插满 100 就插不进去 | 只要内存够就能一直插 |

一句话选型：**查得多就用顺序表，改得多、且改的位置已知就用链表。** 课程代码里链表常常显得「更麻烦还没更快」，原因就是所有操作都得先 O(n) 定位，链表 O(1) 插入的优势只有在你手里已经攥着那个前驱指针时才兑现——比如刚才反复删第 1 位、或者已知前驱的双向链表。

## 五、两份代码里值得注意的细节

整理过程中发现几处可以优化的地方，不影响 main 跑出正确结果，但值得记一笔。

### 1. 链表的 `INIT_List` 申请了 100 个结点

```c
L=(Linklist)malloc(INITLISTSIZE*sizeof(Lnode));   // 申请 100 个结点的空间
```

头结点只需要**一个**：

```c
L=(Linklist)malloc(sizeof(Lnode));
```

现在的写法浪费了 99 个结点的内存，而且会让人误以为链表也有「容量上限 100」——并没有，后续 `Insert_List` 里每次插入都单独 `malloc(sizeof(Lnode))`，长度完全不受 100 限制。`INITLISTSIZE` 这个宏在链表里其实是多余的，它更像是从顺序表代码里顺手抄过来的。

### 2. 链表没有释放头结点

`main` 里删光了所有数据结点，但头结点本身既没 `free` 也没置空，程序结束前一直占着内存。链表应该补一个销毁操作：

```c
Status DestroyList(Linklist &L)
{
	Linklist p = L, q;
	while(p)
	{
		q = p->next;
		free(p);
		p = q;
	}
	L = NULL;
	return OK;
}
```

### 3. `Insert_List` 的 `i = 0` 会静默插到第 1 位

```c
while(p&&j<i-1)   // i = 0 时 i-1 = -1
```

`j = 0` 不小于 `-1`，所以循环不执行，`p` 停在头结点，判空通过，元素被插到了**第 1 位**。`i = 0` 这种非法位置不但没报错，还被当成了 `i = 1` 处理。对比顺序表就很明显：

```
InsertList_Sq(i=0) 返回=0  length=100    ← 被挡住了
```

顺序表的 `if(i<1||i>L.length+1)` 挡住了 `i=0`，链表的 `if(!p||j>i-1)` 挡不住。**判断「位置非法」时，光判 `j > i-1` 不够，还得补一个 `i < 1`。**

### 4. 越界时引用参数 `e` 保留旧值

前面实测的 `GetElem(0) r=0 e=3` 已经说明：失败时 `e` 不会被清零或赋值。调用方必须严格检查 `Status` 返回值，否则会拿着上一次的数据当真。更稳妥的写法是让输出参数在失败时也有明确语义（比如置 0）。

### 5. `main` 返回 `1` / `返回 OK`

```c
printf("List length:%d\n", list.length);
return 1;        // 顺序表：main 返回 1
```

`main` 的返回值会作为进程退出码传给操作系统，**0 才是「正常结束」**。返回 1 会让调用方（shell、CI、批处理）以为程序出错。链表那份同理，`return OK` 也是 `return 1`。改成 `return 0;` 更规范。

### 6. 函数命名不统一

| 操作 | 顺序表 | 链表 |
|------|--------|------|
| 初始化 | `InitList_Sq` | `INIT_List` |
| 取值 | `GetElemList_Sq` | `GetElem` |
| 插入 | `InsertList_Sq` | `Insert_List` |
| 删除 | `DeleteList_Sq` | `Delete_List` |

顺序表用 `_Sq` 后缀区分结构，链表却用大小写和下划线混搭。加上链表缺了 `DestroyList` / `ClearList`，两个文件的接口风格不一致。真要合起来用，最好统一成 `InitList_L` / `GetElem_L` / `InsertList_L` / `DeleteList_L` 这样对称的命名。

### 7. 重复初始化会内存泄漏

```c
InitList_Sq(list);   // 已经初始化过
InitList_Sq(list);   // 原 elem 的地址丢了，泄漏 100 个 int
```

`InitList_Sq` 里的 `malloc` 没有先检查 `L.elem` 是否已经指向一块内存。稳妥的做法是先判断再分配，或者在文档里明确「每个表只初始化一次」。另外 `DestroyList_Sq` 之后紧接着调 `ClearList_Sq` 是安全的（`!L.elem` 会直接返回 `ERROR`），这个防御写得不错。

## 六、结语

两份代码加起来不到 220 行，却几乎覆盖了线性表的全部考点：`i-1` 的下标偏移、`length` 与 `listsize` 的分工、插入删除相反的循环方向、链表「先接后断」的两句指针赋值、头结点带来的判空统一、以及两种结构在复杂度上的此消彼长。

真正让我体会最深的一点是：**顺序表和链表的复杂度表背下来很容易，但边界条件写对很难。** 上面那 7 个细节里，除了第 1、2 条是风格问题，其余 5 条全和边界有关——`i=0`、表满、空表、越界、输出参数的残留值。考试考的、面试问的、线上出 bug 的，基本都是这些地方。

所以建议和我一样：把每个操作都像第二节的插入、第三节的删除那样，**拿一张纸画一遍元素搬移或指针指向**，再故意把 `for` 的方向、把两句指针赋值调换一下，亲手看看结果怎么坏掉。破坏一次比看十遍正确代码管用。
