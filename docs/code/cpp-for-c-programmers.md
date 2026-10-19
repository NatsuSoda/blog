---
title: 从 C 到 C++：初学者该注意什么，两者差在哪里
description: 为有 C 基础的初学者梳理 C++ 的核心新概念：引用、cin/cout、string、new/delete、类与结构体的差异，附上 7 个常见陷阱和逐条对照表，帮你少走弯路。
date: 2026-10-19
tags:
  - C++
  - C语言
  - 编程入门
  - 代码笔记
---

# 从 C 到 C++：初学者该注意什么，两者差在哪里

很多人学 C++ 的路线是「先学 C，再转 C++」。这条路线有一个隐患：**C 学得越顺手，转 C++ 时越容易用 C 的思路写 C++ 代码**——能跑，但没用到 C++ 提供的东西，等遇到类、引用、异常这些特性时反而更懵。

这篇文章不是从零讲语法，而是专门盯着「从 C 切换过来时容易卡壳的地方」：哪些东西在 C++ 里换了写法，哪些是 C++ 独有的，哪些地方手感和 C 很像但藏着坑。

## 一、先说清楚：C++ 兼容 C，但不是 C 的超集

一个常见误解是「C++ 就是带类的 C」。这句话在入门阶段方向上没错，但有两处需要纠正：

**第一，几乎所有合法的 C 代码都能在 C++ 编译器里编译**，但有少数例外。最常见的一条是隐式类型转换：

```c
// C 里合法，C++ 里报错
void *p = malloc(sizeof(int));
int *q = p;   // C 允许 void* 隐式转为 int*，C++ 不允许
```

C++ 的类型检查更严，`void*` 必须显式转换：

```cpp
int *q = (int *)malloc(sizeof(int));  // C++ 里必须加强制转换
```

**第二，C++ 引入了大量 C 没有的机制**——引用、命名空间、类、模板、异常处理、STL 标准库。这些不是可选项，而是 C++ 代码日常就在用的东西，绕开它们只会越写越别扭。

下面逐个拆解最关键的几处差异。

## 二、头文件与命名空间：第一行代码就不一样

C 的标准头文件是 `<stdio.h>`、`<string.h>`、`<stdlib.h>`，文件名带 `.h`。C++ 的标准库头文件**去掉了 `.h`**，变成 `<iostream>`、`<string>`、`<vector>`：

```cpp
// C 风格
#include <stdio.h>
#include <string.h>

// C++ 风格
#include <iostream>
#include <string>
```

C++ 里还有一套兼容 C 的头文件，`<stdio.h>` 对应的 C++ 版本是 `<cstdio>`，`<string.h>` 对应 `<cstring>`。两种写法都能用，但混用容易出问题，**同一个项目里选一套就好**，C++ 项目推荐用不带 `.h` 的版本。

除了头文件，C++ 还引入了**命名空间（namespace）**。标准库里所有的东西都放在 `std` 命名空间下，用的时候要加 `std::` 前缀：

```cpp
#include <iostream>

int main() {
    std::cout << "hello" << std::endl;
    return 0;
}
```

写全 `std::` 每次都要多打几个字，所以初学代码里常见一行 `using namespace std;`，这样就能省略前缀。这个写法在小程序里无所谓，真正的工程项目里为什么要避开它，第五节会专门讲。

## 三、输入输出：cin/cout vs scanf/printf

这是最直观的差异。C 用 `scanf`/`printf`，C++ 多了 `cin`/`cout`。

### `cout` 输出

```cpp
#include <iostream>
using namespace std;

int a = 42;
double b = 3.14;
string s = "hello";

// C 的写法
printf("%d %.2f %s\n", a, b, s.c_str());

// C++ 的写法
cout << a << " " << fixed << setprecision(2) << b << " " << s << endl;
```

`cout` 用 `<<` 运算符把多个值串起来输出，不用记格式字符串 `%d`、`%f`。输出整数直接传整数，输出字符串直接传字符串，类型由编译器推断。

### `cin` 输入

```cpp
int n;
string name;

// C 的写法
scanf("%d %s", &n, name_buf);

// C++ 的写法
cin >> n >> name;
```

`cin` 用 `>>` 接收输入，同样不用写格式字符串，也不用手动取地址 `&`（`string` 和内置类型都直接传变量名）。

### 两者可以混用吗

技术上可以，但默认情况下会有性能问题——C++ 的流和 C 的流会同步，每次 `cin`/`cout` 都要刷新 C 的缓冲区。竞赛代码里常见这两行优化：

```cpp
ios::sync_with_stdio(false);
cin.tie(nullptr);
```

加了这两行，`cin`/`cout` 的速度就能接近 `scanf`/`printf`。但加了之后就不能再混用 C 和 C++ 的 IO 了，否则输出顺序会乱。

## 四、引用：C++ 独有的别名机制

**引用是 C++ 里最需要认真理解的概念**，没有之一。C 里只有指针，C++ 同时有指针和引用，初学者很容易把两者搞混。

### 引用是变量的别名

```cpp
int a = 10;
int &ref = a;   // ref 是 a 的引用，即 a 的别名

ref = 20;       // 修改 ref 等于修改 a
cout << a;      // 输出 20
```

`ref` 和 `a` 指向同一块内存，操作任何一个都影响另一个。区别在于：**引用必须在声明时初始化，而且之后不能再绑定到别的变量**。

```cpp
int a = 1, b = 2;
int &ref = a;    // 绑定到 a
ref = b;         // 这不是让 ref 改绑 b，而是把 b 的值赋给 a
// 此时 a == 2，ref 仍然绑定到 a，不是 b
```

### 引用传参 vs 指针传参

C 里要修改函数外的变量，必须传指针：

```c
// C：传指针
void swap(int *a, int *b) {
    int tmp = *a;
    *a = *b;
    *b = tmp;
}
swap(&x, &y);
```

C++ 可以用引用传参，写法更干净：

```cpp
// C++：传引用
void swap(int &a, int &b) {
    int tmp = a;
    a = b;
    b = tmp;
}
swap(x, y);   // 调用时不用取地址
```

效果完全一样，但调用方不用写 `&x`，函数体里不用 `*` 解引用。前面那篇顺序表和链表的代码里，`SqList &L`、`ElemType &e` 用的就是这种写法——既能在函数内修改外部变量，又比指针写起来简洁。

### `&` 在 C++ 里有两种含义

这是常见的混淆点：

```cpp
int a = 5;
int &ref = a;    // 声明语句里的 & → 引用声明，ref 是 a 的别名
int *p = &a;     // 表达式里的 & → 取地址运算符，和 C 一样
```

一个简单的区分方法：`&` 出现在**变量类型声明里**（紧跟类型名或变量名前），是引用；出现在**表达式里**（紧跟变量名前，没有类型），是取地址。

## 五、字符串：`std::string` vs `char[]`

C 里字符串是字符数组 `char[]`，结尾用 `\0` 标记，操作靠 `<string.h>` 里的函数（`strcpy`、`strlen`、`strcat`……）。C++ 引入了 `std::string`，让字符串变成了一个真正的对象。

```cpp
#include <string>
using namespace std;

string s1 = "hello";
string s2 = "world";

// 拼接：直接用 +
string s3 = s1 + " " + s2;    // "hello world"

// 长度
cout << s1.length();           // 5

// 比较：直接用 ==
if (s1 == "hello") { ... }

// 子串
string sub = s1.substr(1, 3);  // "ell"

// 查找
size_t pos = s3.find("world"); // 6
```

C 里的字符串比较要用 `strcmp`，拼接要用 `strcat` 并且手动保证目标缓冲区够大，`std::string` 全部内置了，而且不会溢出。

### `string` 和 `char*` 之间的转换

和 C 的库函数（比如 `printf`、`fopen`）交互时，需要把 `string` 转成 C 风格的字符串：

```cpp
string filename = "data.txt";
FILE *f = fopen(filename.c_str(), "r");   // .c_str() 返回 const char*
```

反过来，`char*` 可以直接赋值给 `string`：

```cpp
const char *cstr = "hello";
string s = cstr;   // 自动转换
```

## 六、内存管理：`new`/`delete` vs `malloc`/`free`

C++ 有自己的内存分配运算符，不再需要 `malloc`/`free`（虽然 C++ 里仍然可以用）。

### 基础用法

```cpp
// C：malloc + free
int *p = (int *)malloc(sizeof(int));
*p = 42;
free(p);

// C++：new + delete
int *p = new int;
*p = 42;
delete p;

// 带初始值
int *p = new int(42);    // *p == 42

// 数组
int *arr = new int[10];
arr[0] = 1;
delete[] arr;    // 数组必须用 delete[]，不是 delete
```

`new` 和 `malloc` 的核心区别不只是语法：**`new` 会调用构造函数，`delete` 会调用析构函数**。这一点在你开始用类的时候会非常重要——用 `malloc` 分配的类对象，构造函数不会执行，对象状态是未定义的。

一个常见错误是数组分配和释放不匹配：

```cpp
int *arr = new int[10];
delete arr;       // ❌ 应该用 delete[]，这里是未定义行为
delete[] arr;     // ✅
```

## 七、类与结构体：C++ 里 struct 也是类

C 的 `struct` 只能放数据。C++ 的 `struct` 和 `class` 功能几乎一样，都可以有成员函数、构造函数、析构函数。唯一区别是**默认访问权限**：`struct` 的成员默认是 `public`，`class` 的成员默认是 `private`。

### 最简单的类

```cpp
class Point {
public:           // public 以下的成员外部可访问
    int x, y;

    // 构造函数：对象创建时自动调用
    Point(int x, int y) : x(x), y(y) {}

    // 成员函数
    double distance() {
        return sqrt(x * x + y * y);
    }
};

// 使用
Point p(3, 4);
cout << p.distance();   // 5.0
```

### 构造函数和析构函数

**构造函数**在对象创建时自动执行，用来初始化成员变量。**析构函数**在对象销毁时自动执行，用来释放资源（比如 `delete` 掉构造函数里 `new` 的内存）：

```cpp
class Buffer {
public:
    int *data;
    int size;

    Buffer(int n) {         // 构造函数
        size = n;
        data = new int[n];
    }

    ~Buffer() {             // 析构函数，函数名前加 ~
        delete[] data;
    }
};

{
    Buffer buf(100);        // 构造函数调用，分配 100 个 int
    buf.data[0] = 1;
}   // 出作用域，析构函数自动调用，delete[] 释放内存
```

C 里资源管理是手动的——`malloc` 之后要记得 `free`。C++ 的析构函数让资源管理可以绑定在对象生命周期上，对象消亡时自动清理，这是 C++ 资源管理的核心思想（RAII）。

## 八、初学者容易踩的 7 个坑

### 1. `using namespace std` 的副作用

初学代码几乎人人都写 `using namespace std;`，短程序里没问题。但它把 `std` 里几百个名字都引入了全局，这意味着你自己起的变量名可能和标准库冲突：

```cpp
using namespace std;

int count = 0;   // std 里也有 count 函数，这里会遮蔽它
```

碰巧重名时编译器报一个莫名其妙的错误，很难排查。工程代码的做法是**只引入需要的名字**：

```cpp
using std::cout;
using std::cin;
using std::string;
```

或者每次都写全 `std::cout`，多打几个字，但绝不出岔子。

### 2. `endl` 比 `"\n"` 慢

```cpp
cout << "hello" << endl;    // endl 输出换行并刷新缓冲区
cout << "hello" << "\n";    // \n 只输出换行
```

每次 `endl` 都会把缓冲区强制刷到终端。循环里大量输出时，`endl` 比 `"\n"` 慢几倍甚至几十倍。除非确实需要立刻看到输出（调试某些崩溃场景），否则统一用 `"\n"` 就好。

### 3. 引用必须在声明时初始化

```cpp
int &ref;        // ❌ 编译错误，引用不能「先声明后绑定」
int a = 5;
int &ref = a;    // ✅
```

指针可以先声明再赋值，引用不行。函数参数里的引用是例外——绑定发生在调用时，调用前不需要显式初始化。

### 4. `&` 的两种含义容易看走眼

```cpp
void foo(int &x) { x = 10; }   // 声明里的 & → 引用参数
int a = 5;
foo(a);           // 调用时不加 &，直接传变量名

int *p = &a;      // 表达式里的 & → 取地址
```

刚从 C 过来的人看到 `foo(a)` 会以为是值传递，看到 `int &x` 才反应过来是引用。养成习惯：看函数声明确认有没有 `&`，决定调用方式。

### 5. `delete[]` 和 `delete` 不能混用

```cpp
int *single = new int;
int *arr    = new int[10];

delete   single;   // ✅
delete[] arr;      // ✅

delete   arr;      // ❌ 未定义行为，可能崩溃，可能静默内存泄漏
delete[] single;   // ❌ 同上
```

`new[]` 分配的内存只能用 `delete[]` 释放，反之亦然。运行时不会给你报错，但行为是未定义的——最常见的表现是程序跑到某处莫名崩溃，很难定位。

### 6. 构造函数不能手动调用

```cpp
class Foo {
public:
    Foo() { cout << "构造\n"; }
};

Foo obj;
obj.Foo();   // ❌ 编译错误，构造函数不能这样调用
```

构造函数由系统在对象创建时调用，程序员不能手动触发。如果需要「重置对象状态」，应该单独写一个 `reset()` 成员函数。

### 7. 类对象的赋值和复制不是内存拷贝

```cpp
Buffer a(10);
Buffer b = a;   // 默认的拷贝构造：把 a.data 的指针值复制给 b.data
// 现在 a.data 和 b.data 指向同一块内存
```

默认的拷贝构造函数做的是逐成员浅拷贝——对于含指针的类，两个对象会共享同一块 `new` 出来的内存。一个对象析构时 `delete` 了，另一个再访问就是悬空指针。这个问题叫**浅拷贝陷阱**，解决方法是自定义拷贝构造函数（或者直接用智能指针）。

## 九、C 与 C++ 对照总结

| 对比项 | C | C++ |
|--------|---|-----|
| 标准输入输出 | `scanf` / `printf` | `cin` / `cout`（也可用 C 的） |
| 标准头文件 | `<stdio.h>` | `<iostream>`（C 的用 `<cstdio>`）|
| 字符串 | `char[]` + `<string.h>` | `std::string` |
| 内存分配 | `malloc` / `free` | `new` / `delete`（构造/析构）|
| 函数传参修改外部变量 | 指针 `int *p` | 引用 `int &r`（或指针）|
| 结构体 | 只有数据 | 可有成员函数、构造/析构函数 |
| 类型检查 | 较宽松（`void*` 隐式转换）| 严格（必须显式转换）|
| 命名空间 | 无 | `namespace`，标准库在 `std::` 下 |
| 函数重载 | 不支持 | 支持（同名函数，参数不同）|
| 默认参数 | 不支持 | 支持 |
| `bool` 类型 | C99 才有 `<stdbool.h>` | 原生支持，`true` / `false` |

## 十、学习路径建议

有 C 基础再学 C++，可以按这个顺序推进，不容易绕进去：

**第一步，掌握引用**。这是 C++ 里用得最广的新特性，而且是学类、学 STL 的前置知识。把「引用是别名，不是指针，声明时必须初始化，不能改绑」这几条搞透。

**第二步，换掉 C 的 IO 和字符串**。`cin`/`cout` 和 `std::string` 是最直接的使用体验提升，能帮你把注意力从「格式字符串」和「缓冲区」上解放出来。

**第三步，学类的基础**：构造函数、析构函数、成员函数、`public`/`private` 访问控制。不用急着学继承和多态，先把「怎么把数据和操作数据的函数打包在一起」这件事理解清楚。

**第四步，了解 STL 的 `vector`、`string`、`map`**。这三个容器覆盖了 80% 的日常场景，用熟了你会发现 C 里手写数组和链表的很多事情已经被封装好了。

C++ 的特性很多，不需要一次全学。先把这篇里的核心差异搞清楚，能写出正确的 C++ 代码，再去碰模板、异常处理、移动语义这些进阶话题——那时候你会有一套清晰的上下文去理解它们，而不是死记硬背。
