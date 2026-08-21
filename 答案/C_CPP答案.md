# C / C++ 答案

## CPP-K-001 `volatile` 在嵌入式中的作用是什么？

### 面试口述版
`volatile` 告诉编译器：该对象的值可能在当前执行流之外发生变化，因此每次访问都应按语言和编译器规则实际读取/写回，而不能把之前读到的值长期缓存到寄存器中。嵌入式中常见于内存映射外设寄存器、ISR 与主循环共享的简单标志等。但 `volatile` **不保证原子性，也不等于线程同步**。

### 高频追问
- `volatile` 能解决竞态条件吗？不能。
- `volatile` 和 `const` 能一起用吗？可以，例如只读状态寄存器。
- ISR 与任务共享 32 位变量一定安全吗？取决于架构、对齐和访问宽度，不能只凭 `volatile` 下结论。

---

## CPP-C-001 实现一个环形缓冲区的核心读写逻辑

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>

#define RB_CAPACITY 8

typedef struct {
    uint8_t data[RB_CAPACITY];
    size_t head;  // 下一次写入位置
    size_t tail;  // 下一次读取位置
    size_t count; // 当前元素数
} RingBuffer;

bool rb_push(RingBuffer *rb, uint8_t v) {
    if (!rb || rb->count == RB_CAPACITY) return false;
    rb->data[rb->head] = v;
    rb->head = (rb->head + 1) % RB_CAPACITY;
    rb->count++;
    return true;
}

bool rb_pop(RingBuffer *rb, uint8_t *out) {
    if (!rb || !out || rb->count == 0) return false;
    *out = rb->data[rb->tail];
    rb->tail = (rb->tail + 1) % RB_CAPACITY;
    rb->count--;
    return true;
}
```

### 面试要点
先说明“满/空如何区分”。上面使用 `count`，写起来最清楚；工程上还可采用牺牲一个槽位、单调递增读写索引等方式。

### 追问
- ISR 写、Task 读时是否线程安全？
- `%` 能不能优化掉？
- 如何做成无锁 SPSC？

---

## CPP-C-002 指定区间链表反转

```cpp
struct ListNode {
    int val;
    ListNode *next;
};

ListNode* reverseBetween(ListNode* head, int left, int right) {
    if (!head || left == right) return head;

    ListNode dummy{0, head};
    ListNode *pre = &dummy;

    for (int i = 1; i < left; ++i) {
        pre = pre->next;
    }

    ListNode *cur = pre->next;
    for (int i = 0; i < right - left; ++i) {
        ListNode *move = cur->next;
        cur->next = move->next;
        move->next = pre->next;
        pre->next = move;
    }
    return dummy.next;
}
```

### 核心
dummy 节点用于统一 `left == 1` 的边界；时间复杂度 O(n)，额外空间 O(1)。

---

## CPP-D-001 修复有数据竞争的环形缓冲区

### 训练目标
如果一个 ISR 修改 `head`，任务修改 `tail`，不要看到 `volatile` 就认为安全。先判断：
1. 是否 SPSC；
2. 单次读写是否原子；
3. 内存可见性与重排序；
4. 满/空判定是否同时依赖多个共享变量。

### 优化方向
在 MCU ISR + 单任务的 SPSC 场景，可设计为各自只写自己的索引，必要时使用短临界区或平台原子/内存屏障。多生产者/多消费者则不能直接套用。

---

## CPP-O-001 优化频繁 `malloc/free` 的数据处理链

### 建议顺序
1. 先确认最大并发对象数和最大数据尺寸；
2. 优先静态分配或固定块内存池；
3. 可复用对象避免重复构造/释放；
4. 若必须动态分配，监控峰值、失败率和碎片；
5. 实时路径避免不可预测的分配延迟。

---

## CPP-C-003 用二级指针创建并返回动态对象

### 30~90秒面试版
函数参数按值传递，若被调函数需要改变调用者持有的`int *`，必须接收它的地址，即`int **`。实现时先检查输出参数，再申请内存、检查失败，最后初始化对象；成功后所有权转移给调用者，由调用者唯一负责`free`。

```c
#include <stdlib.h>

int create_data(int **out)
{
    if (out == NULL) {
        return -1;
    }

    int *tmp = malloc(sizeof *tmp);
    if (tmp == NULL) {
        return -1;
    }

    *tmp = 100;
    *out = tmp;
    return 0;
}
```

调用者应把输出指针初始化为`NULL`，仅在成功后使用，并在结束时`free`。本实现先完成初始化再写回`*out`，因此失败时不会破坏调用者原值。

### 常见错误
- `int x; *out = &x;`：函数返回后形成悬空指针。
- 只传`int *`再给形参赋值：修改的是形参副本。
- 不检查`out`或`malloc`结果：可能空指针解引用。
- 释放责任不明确：容易重复释放、泄漏或释放后使用。

---

## CPP-D-002 数组、字符串与内存函数综合排错

### 排错顺序
1. 判断对象是数组还是指针，以及是否已发生数组到指针转换。
2. 区分容量、有效长度和终止符所需空间。
3. 判断源/目的区域是否重叠。
4. 核对复制后的对象是否满足下游API前提，例如`printf("%s")`要求可达的`'\0'`。

```c
void clear_bad(int a[16])
{
    memset(a, 0, sizeof a); /* sizeof a是指针大小 */
}
```

可把元素数量作为参数传入：

```c
void clear_ints(int *a, size_t count)
{
    if (a != NULL) {
        memset(a, 0, count * sizeof *a);
    }
}
```

`strncpy(dst, src, sizeof dst)`不能普遍保证字符串终止；`memcpy`面对重叠区域则是未定义行为，应改用`memmove`。

---

## CPP-K-003 限定符与寄存器访问综合辨析

### 30~90秒面试版
`volatile`解决编译器可见性层面的访问优化，不解决并发原子性；`const`限制通过特定左值修改；`static`可能表示静态存储期或内部链接；`extern`声明外部定义。对MMIO寄存器，还必须读取芯片手册的位属性，W1C位不能随意用普通读—改—写。

### 追问检查
- `volatile uint32_t counter; counter++;`为什么仍可能竞态？它是读、修改、写的复合操作。
- `const volatile uint32_t *reg`表示什么？通过该指针软件不可写，但值可能异步变化；指针本身仍可改指向。
- 文件内`static int g;`能否被其他`.c`的`extern int g;`访问？不能，定义具有内部链接。

---

## CPP-K-004 结构体对齐、字节序与协议布局

```c
struct Test2 {
    char  a;
    short b;
    int   c;
    char  d;
};
```

在常见`char/short/int`对齐为`1/2/4`的ABI下，偏移依次为`0/2/4/8`，尾部补齐后大小为12。但面试时要声明ABI假设；语言本身不固定这些具体偏移。

结构体内存镜像不适合作为稳定线协议，因为两端可能在填充、ABI、字节序和版本上不同。正确做法是规定字段宽度与线序并逐字段编码；接收端校验长度、版本和范围后再构造本地对象。

---

## CPP-C-004 设计带上下文的驱动回调接口

```c
typedef void (*UartRxCallback)(const unsigned char *data,
                               size_t length,
                               void *context);

typedef struct {
    UartRxCallback on_rx;
    void *context;
} UartCallbacks;
```

驱动触发时先判空，再以约定参数调用。注册时还要写清：回调在ISR还是任务上下文执行、`data`的有效期、能否重入、是否允许阻塞。若在ISR执行，应只做最小工作并把后续处理延后到任务。

---

## CPP-D-003 判断预处理、编译、链接与运行期错误

| 现象 | 阶段 | 首要检查 |
|---|---|---|
| `xxx.h: No such file or directory` | 预处理/编译 | include路径与文件名 |
| `expected ';'`、类型不匹配 | 编译 | 语法、声明和类型 |
| `undefined reference` | 链接 | 定义是否参与构建、符号名与库顺序 |
| `multiple definition` | 链接 | 是否在头文件重复定义普通全局对象 |
| 构建成功但HardFault/随机异常 | 运行期 | 指针、栈、数组边界、控制流和硬件状态 |

include guard只能抑制同一翻译单元的重复包含；它不能把写在头文件中的外部链接对象定义自动变成“全程序唯一”。

---

## CPP-D-004 寄存器清位时为什么不能使用逻辑非

### 原始错误

```c
reg &= !(1U << 5);
```

### 正确答案

`1U << 5`是非零值，逻辑非`!`作用后得到整数0，因此原式等价于：

```c
reg &= 0U;
```

它会把整个寄存器清零。清除第5位需要用按位取反`~`构造除目标位外全为1的掩码：

```c
reg &= ~(1U << 5);
```

### 四种基本操作

```c
reg |=  (1U << n);          // 置1
reg &= ~(1U << n);          // 清0
reg ^=  (1U << n);          // 翻转
bit = (reg >> n) & 1U;      // 读取
```

### 工程边界

上述写法属于普通读—改—写。对W1C、只写、读清零或具有其他副作用的寄存器，必须按芯片手册规定访问，不能直接套用。

---

## CPP-K-005 typedef与define声明辨析

### 原始问题

```c
typedef char *PSTR;
PSTR a, b;

#define PSTR2 char *
PSTR2 c, d;
```

### 正确答案

- `PSTR`是`char *`的类型别名，因此`a`和`b`都是`char *`。
- `PSTR2`只是文本宏。第二个声明展开为`char *c, d;`，因此`c`是`char *`，`d`是`char`。
- `c`不是`char **`。

### 错误根因

宏没有创造新类型；分析宏声明时应先做文本展开，再按C声明符分别解析每个变量。`typedef`更适合复杂指针和函数指针类型，因为它保留类型语义并减少声明歧义。
