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
