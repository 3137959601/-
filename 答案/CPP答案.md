# C++：答案与解析

每个答案标题下直接给出解析，不再跳转其他题库。题号用于唯一绑定；“返回例题”回到原题，不要求额外打开答案索引。

## CPP-D-012

**引用绑定、const引用与生命周期**

[返回例题](../嵌入式秋招八股文_CPP.md#q-cpp-d-012)

`int&`只能绑定可修改左值，不能绑定`const int`或临时值；`const int&`可绑定普通对象、const对象和临时值。const限制的是通过该引用写入，普通底层对象仍可经其他路径修改。返回局部自动变量引用会悬空；static对象不会悬空，但会共享状态并影响可重入性。

### 本题逐项解析

r1/r4/r5/r6 合法，r2/r3 非法；a=9 后 r4 观察到9。r6 直接绑定临时值可在这里延寿，不能推广为返回引用也延寿。

### 你曾经的易错点

曾认为`int&`可绑定 const 对象和临时值，并把 const 引用理解成底层对象不能变化。

---

## CPP-D-013

**构造析构顺序与new数组配对**

[返回例题](../嵌入式秋招八股文_CPP.md#q-cpp-d-013)

同一作用域按定义顺序构造、逆序析构；派生对象按 Base→Derived 构造，按 Derived→Base 析构。数组分配必须写成`T* p = new T[n]; delete[] p;`；`delete p[]`语法错误，`new[]`配`delete`则是未定义行为。

### 本题逐项解析

输出 C C D D，前一个 D 对应 b，后一个对应 a。new A[2] 应 delete[] p；delete p[] 是语法错误；malloc 指针应 free，但对非平凡类还必须先合法构造对象，不能仅申请字节后当完整 T 使用。

### 你曾经的易错点

把析构顺序说成与构造相同，写过`delete p[]`，并把`new[]/delete`混用只归结为泄漏。

---

## CPP-D-014

**成员初始化顺序**

[返回例题](../嵌入式秋招八股文_CPP.md#q-cpp-d-014)

成员按类内声明顺序初始化，而非初始化列表顺序。若`a`先声明、`b`后声明，则`: b(10), a(b)`仍先初始化`a`，此时读取尚未初始化的`b`有问题。应调整声明依赖并按声明顺序书写初始化列表。

### 本题逐项解析

a 先初始化，读取尚未初始化的 b，不能预测为10。可直接写 X():a(10),b(10){}；若 a 依赖 b，则调整声明顺序为 int b; int a; 并写 :b(10),a(b)。

### 你曾经的易错点

认为成员按初始化列表书写顺序初始化。

---

## CPP-D-015

**new、malloc与分配失败**

[返回例题](../嵌入式秋招八股文_CPP.md#q-cpp-d-015)

`new`分配并构造，`delete`析构并释放；`malloc/free`只管理原始内存。普通 throwing `new`失败时抛`std::bad_alloc`，不是返回该对象；`new(std::nothrow)`失败返回`nullptr`，`malloc`失败返回空指针。

### 本题逐项解析

前两项返回 T* 且构造；malloc 返回 void*，只给原始存储。普通 new 分配失败抛 bad_alloc，nothrow 分配失败空指针，malloc 失败空指针。placement 构造后的对象用 p->~T() 结束生命周期，再 free 原存储；构造失败也必须 free 原存储。nothrow 不吞掉构造异常。

### 你曾经的易错点

把构造/析构能力说反，并说普通`new`失败会“返回`bad_alloc`”。

---

## CPP-D-016

**拷贝、移动与资源所有权**

[返回例题](../嵌入式秋招八股文_CPP.md#q-cpp-d-016)

默认复制裸资源指针会让两个对象共享同一地址。资源类需遵守 Rule of Three/Five，或更优地使用标准 RAII 成员实现 Rule of Zero。`T&&`是右值引用；`std::move`只转换值类别。移动接管资源后应让源对象保持有效、可析构、可重新赋值，通常将源裸指针置空以避免双重所有权和 double free。

### 本题逐项解析

以下 C++17 类可直接用于语法检查；拷贝先申请独立空间，失败时尚未交换资源，所以目标保持不变。

```cpp
#include <algorithm>
#include <cstddef>
#include <utility>
class Buffer {
    unsigned char* data_ = nullptr;
    std::size_t size_ = 0;
public:
    explicit Buffer(std::size_t n = 0)
        : data_(n ? new unsigned char[n]{} : nullptr), size_(n) {}
    ~Buffer() { delete[] data_; }
    Buffer(const Buffer& b) : Buffer(b.size_) {
        if (size_) std::copy_n(b.data_, size_, data_);
    }
    Buffer& operator=(const Buffer& b) {
        if (this != &b) { Buffer tmp(b); swap(tmp); }
        return *this;
    }
    Buffer(Buffer&& b) noexcept
        : data_(std::exchange(b.data_, nullptr)),
          size_(std::exchange(b.size_, 0)) {}
    Buffer& operator=(Buffer&& b) noexcept {
        if (this != &b) {
            delete[] data_;
            data_ = std::exchange(b.data_, nullptr);
            size_ = std::exchange(b.size_, 0);
        }
        return *this;
    }
    void swap(Buffer& b) noexcept {
        std::swap(data_, b.data_);
        std::swap(size_, b.size_);
    }
};
```

移动只转移地址与长度；std::move 本身不执行这些动作。实际业务若使用 std::vector<unsigned char> 成员，通常可直接采用 Rule of Zero。

### 你曾经的易错点

把`T&&`说成引用的引用，认为移动后源对象不能析构，并低估源指针置空对所有权安全的作用。

---

## CPP-D-017

**静态类型、动态绑定与基类引用**

[返回例题](../嵌入式秋招八股文_CPP.md#q-cpp-d-017)

`Base*`的静态类型决定可见接口，实际对象的动态类型决定虚函数最终重写版本。`Base& r = derived`是引用而非指针，同样保留多态；若函数不是 virtual，则不会发生运行时动态分派。

### 本题逐项解析

依次为1、4、1、4。非虚 f 按 Base 静态接口绑定；虚 g 按 Derived 动态类型分派。引用不会创建一个 Base 副本。

### 你曾经的易错点

困惑`Base*`为何调用派生重写，并把`Base&`称为指针。

---

## CPP-D-018

**对象切片与浅拷贝**

[返回例题](../嵌入式秋招八股文_CPP.md#q-cpp-d-018)

`Base b = derived`创建真正的 Base 对象，只复制派生对象中的基类子对象，属于对象切片。浅拷贝讨论的是资源成员（如指针）只复制地址。避免切片应按引用或指针传递多态对象。

### 本题逐项解析

前两项切片，第三项保留多态，第四项是指针资源浅拷贝，不因它是拷贝就自动构成对象切片。按引用/指针传递解决切片；深拷贝、禁止复制或 RAII 解决资源所有权，二者不能互相替代。

### 你曾经的易错点

把`Base b = derived`解释成浅拷贝。

---

## CPP-D-019

**虚析构与派生析构顺序**

[返回例题](../嵌入式秋招八股文_CPP.md#q-cpp-d-019)

通过非虚析构基类指针删除派生对象属于未定义行为，不能描述为“只删除Base部分”。用于多态删除的基类应声明`virtual ~Base() = default;`。完整派生对象销毁时，先执行 Derived 析构，再销毁成员，最后执行 Base 析构。

### 本题逐项解析

修复为 virtual ~Base()=default。顺序是 Derived 析构函数体 → m2 析构 → m1 析构 → Base 析构函数体（随后再销毁 Base 的成员）。非虚基类删除派生对象不能预测固定输出。

### 你曾经的易错点

把`delete Base*`的后果说成“只删除Base部分”，并答反派生对象析构顺序。

---

## CPP-D-020

**shared_ptr计数与weak_ptr循环**

[返回例题](../嵌入式秋招八股文_CPP.md#q-cpp-d-020)

进入内部作用域复制`shared_ptr`会增加强引用计数，副本离开作用域后计数随即减少；最后一个强所有者销毁或reset时自动释放对象，不能手工delete托管裸指针。双向共享造成循环引用时，把非拥有方向改为`weak_ptr`，访问前用`lock()`检查。

### 本题逐项解析

p 创建后强计数1，w 不增加它；q 存在时2，q 离开后1，p.reset 后0，对象销毁，w.lock() 返回空。让反向观察边使用 weak_ptr，访问时 lock 得到暂时强所有者；不得再手工 delete p.get()。

### 你曾经的易错点

误认为内部作用域结束后引用计数不下降，并认为需要手工delete托管对象。

---

## CPP-D-021

**vector迭代器失效**

[返回例题](../嵌入式秋招八股文_CPP.md#q-cpp-d-021)

扩容时所有旧指针、引用、迭代器失效。无扩容的中间删除通常使删除点及其后失效，删除点之前仍有效。失效对象不能靠`++/--`恢复；循环删除应写`it = v.erase(it)`。

### 本题逐项解析

不扩容的 erase 后只有 i0 保留有效；i1、i2、旧 ie 都失效。随后发生重分配时全部旧迭代器失效。

```cpp
for (auto it = v.begin(); it != v.end(); ) {
    if (*it % 2 == 0) it = v.erase(it);
    else ++it;
}
```

每轮用新的 v.end()，不能通过 ++ 失效的旧迭代器修复它。

### 你曾经的易错点

认为删除点之前的迭代器也失效，并认为失效迭代器可通过移动恢复。

---

## CPP-D-022

**map与unordered_map复杂度**

[返回例题](../嵌入式秋招八股文_CPP.md#q-cpp-d-022)

`map`有序，单次查找通常 O(log n)；`unordered_map`无序，平均查找 O(1)、最坏 O(n)。这里的 O(1)是按键查找平均复杂度，不是完整遍历；遍历 n 个元素必为 O(n)。

### 你曾经的易错点

把unordered_map按键平均查找O(1)说成完整遍历O(1)。

---

## CPP-D-023

**线程同步与生命周期综合**

[返回例题](../嵌入式秋招八股文_CPP.md#q-cpp-d-023)

`counter++`不是天然原子，简单单变量计数可用`atomic`，复合不变量用 mutex。条件变量用谓词等待以应对虚假唤醒。`detach`的主要风险是线程访问对象的生命周期不再由`thread`调用者等待保证；捕获局部引用尤其危险。`std::thread`析构时若仍joinable会调用`std::terminate()`，因此必须明确join或detach策略。

### 本题逐项解析

独立计数可用 std::atomic<int> counter{0} 并执行 ++counter，或在同一互斥下做读改写。数据发布不能只改原子标志而忽略其余同步契约。一个可编译的 C++17 等待例子：

```cpp
#include <condition_variable>
#include <mutex>
#include <thread>
int main() {
    std::mutex m;
    std::condition_variable cv;
    bool ready = false;
    int data = 0;
    std::thread worker([&] {
        std::unique_lock<std::mutex> lk(m);
        cv.wait(lk, [&] { return ready; });
        // 此时持锁，可以访问 data。
        (void)data;
    });
    {
        std::lock_guard<std::mutex> lk(m);
        data = 42;
        ready = true;
    }
    cv.notify_one();
    worker.join(); // 使局部对象活到线程结束
}
```

detach 捕获局部引用的修复优先是在线程结束前 join；也可按值转移独立数据并设计独立的结束协议。joinable 的 thread 析构会 terminate。

### 你曾经的易错点

把detach的主要问题只说成“不符合RAII”，没有指出被访问对象的生命周期责任。

---
