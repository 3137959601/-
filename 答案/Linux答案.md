# Linux：答案与解析

每道题标题下直接给出解析；“返回例题”回到原题，不设置二次答案索引。平台相关边界和完整程序均注明适用条件。

## LINUX-Q01

**进程和线程的核心区别是什么？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q01)

**面试答案：**

进程主要提供资源组织和隔离，普通进程各自拥有虚拟地址空间。线程是进程中的执行流，同一进程的线程共享地址空间、全局变量、堆和文件描述符等资源，但分别维护调用栈、寄存器现场和调度状态。共享使线程间交换数据方便，也意味着必须处理并发访问和对象生命周期。

**解析：**

回答应围绕“共享什么、独立什么”，而不是只背“进程是资源单位、线程是调度单位”。进程不是只能有一个线程；线程也不是没有资源，它仍需要自己的执行现场。


---

## LINUX-Q02

**线程有独立栈，其他线程就访问不到吗？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q02)

**面试答案：**

不是。各线程分别维护调用栈，但这些栈仍在同一个进程地址空间中。如果把一个仍然存活的局部对象地址传给其他线程，其他线程可以通过该地址访问它；问题在于必须保证生命周期和同步。原函数返回后还继续使用该局部对象，就是悬空访问风险。

**解析：**

这是本次补充的边界。“独立栈”是在说保存调用现场的区域分别维护；“独立地址空间”才是另一层隔离。


---

## LINUX-Q03

**为什么线程切换通常比进程切换轻？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q03)

**面试答案：**

两者都需要切换执行现场，但同一进程的线程共享地址空间，通常不需要更换整个地址空间上下文。不同进程的切换可能额外涉及页表相关上下文和地址转换缓存的影响，所以线程切换经常更轻。实际开销取决于架构、调度和缓存状态，不能给出绝对倍数。

**解析：**

不能说“切进程会复制整个进程的内存”。那混淆了创建与调度切换。


---

## LINUX-Q04

**fork 为什么会出现两个返回值？子进程从哪里运行？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q04)

**面试答案：**

fork 成功后存在父、子两个进程，它们从 fork 返回处继续执行。父进程得到子进程 PID，子进程得到 0；失败时父进程得到 -1，没有成功创建子进程。返回值用于区分分支，并不表示函数先在父进程里运行完、再从 main 开头运行一次子进程。

**解析：**

父子都有自己的 `pid` 变量及执行现场，所以得到不同值没有矛盾；调度先后不固定。


---

## LINUX-Q05

**fork 的 COW 会不会让父子修改同一全局变量？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q05)

**面试答案：**

普通私有内存不会因此变成共享可写变量。fork 后父子地址空间逻辑独立，COW 让适合共享的私有页面先共享物理存储，写入时再由内核处理分离。因此子进程修改自己的普通全局变量，不改变父进程的那份。显式共享映射则是另一种语义。

**解析：**

“逻辑上独立”说的是程序可见的隔离语义；“物理页先共享”说的是节省资源的实现方法。两者可以同时成立。


---

## LINUX-Q18

**fork私有变量实验**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q18)

原学习包完整示例，面向 Linux/POSIX，不在 Windows 原生终端直接运行。

```c
#define _POSIX_C_SOURCE 200809L
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

/* Ordinary private process data; NOT an IPC shared-memory region. */
static int g = 1;

int main(void)
{
    /* A single-threaded teaching example. Avoid duplicated stdout buffers. */
    if (setvbuf(stdout, NULL, _IONBF, 0) != 0) {
        fputs("cannot configure stdout\n", stderr);
        return EXIT_FAILURE;
    }

    pid_t child = fork();
    if (child < 0) {
        perror("fork");
        return EXIT_FAILURE;
    }

    if (child == 0) {
        g = 2;
        if (printf("child:  pid=%ld g=%d virtual_address=%p\n",
                   (long)getpid(), g, (void *)&g) < 0) {
            _exit(EXIT_FAILURE);
        }
        _exit(EXIT_SUCCESS);
    }

    int status;
    pid_t result;
    do {
        result = waitpid(child, &status, 0);
    } while (result < 0 && errno == EINTR);

    if (result < 0) {
        perror("waitpid");
        return EXIT_FAILURE;
    }
    if (!WIFEXITED(status) || WEXITSTATUS(status) != 0) {
        fputs("child did not finish successfully\n", stderr);
        return EXIT_FAILURE;
    }

    if (printf("parent: pid=%ld g=%d virtual_address=%p\n",
               (long)getpid(), g, (void *)&g) < 0) {
        return EXIT_FAILURE;
    }
    return g == 1 ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

编译运行：`cc -std=c11 -Wall -Wextra fork_private_memory.c -o demo && ./demo`（请将本题代码单独保存到相应文件）。

预期子g=2、父g=1，父输出发生在回收子之后。普通私有数据逻辑独立；printf中的虚拟地址相同不能证明物理页仍共享。setvbuf避免复制的stdio缓冲导致重复输出。


---

## LINUX-VM-009

**COW与真正只读页**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-009)

### 面试回答

按页相关机制处理，必要时复制包含该数据的页面，并调整写入方映射；不是只复制这个int，也不是每写一次复制整个进程。真正只读页本来就没有写权限，不能因为有COW机制就随意变成可写。

### 原因与解析

COW的目标是维持私有数据语义，同时避免不必要的全量复制。写保护是实现这一策略的手段，内核知道这次写入本来是否允许。若已无需与别人共享，处理还可能不需要实际复制。

### 易错边界

“页表当前不可写”与“应用从语义上永远无权写”不是一个判断。 依据：[VM-S2](https://docs.kernel.org/mm/page_tables.html)[VM-S5](https://man7.org/linux/man-pages/man2/fork.2.html)。


---

## LINUX-Q06

**父子内存独立，为什么 read 可能相互影响文件偏移？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q06)

**面试答案：**

因为 fork 后父子有各自的 fd 表，但继承的描述符引用同一个内核打开文件描述对象，文件偏移等状态属于这个底层对象。内存隔离不意味着所有内核资源也完全独立，所以一方读取可能推进双方共享的文件偏移。

**解析：**

分清“fd 整数”“fd 表项”“open file description”。此处只建立层级，文件 I/O 专题再展开操作细节。


---

## LINUX-Q07

**exec 与 fork 有什么区别？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q07)

**面试答案：**

fork 创建子进程；exec 用新程序映像替换当前进程的程序映像，不创建新进程，PID 保持不变。成功 exec 不返回到旧代码，因此常在子进程里 exec、父进程继续管理并 waitpid。exec 后的代码通常用来处理执行失败。

**解析：**

`exec` 是接口族的称呼；不要求把所有变体背完。重点理解“创建一个进程”与“让现有进程运行另一程序”。


---

## LINUX-Q19

**fork/exec/wait完整示例**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q19)

原学习包完整示例，面向 Linux/POSIX，不在 Windows 原生终端直接运行。

```c
#define _POSIX_C_SOURCE 200809L
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void)
{
    /* Single-threaded parent. The child will replace its program with echo. */
    pid_t child = fork();
    if (child < 0) {
        perror("fork");
        return EXIT_FAILURE;
    }

    if (child == 0) {
        execl("/bin/echo", "echo", "child: now running /bin/echo", (char *)NULL);
        /* Reached only on an ordinary exec failure. */
        perror("execl");
        _exit(127);
    }

    int status;
    pid_t result;
    do {
        result = waitpid(child, &status, 0);
    } while (result < 0 && errno == EINTR);

    if (result < 0) {
        perror("waitpid");
        return EXIT_FAILURE;
    }
    if (WIFEXITED(status)) {
        int code = WEXITSTATUS(status);
        printf("parent: child_pid=%ld exit_code=%d\n", (long)child, code);
        return code;
    }
    if (WIFSIGNALED(status)) {
        fprintf(stderr, "child terminated by signal %d\n", WTERMSIG(status));
    }
    return EXIT_FAILURE;
}
```

编译运行：`cc -std=c11 -Wall -Wextra fork_exec_wait.c -o demo && ./demo`（请将本题代码单独保存到相应文件）。

exec成功不回到旧程序；失败后子用_exit(127)，避免继续走父进程逻辑。父循环处理EINTR并用状态宏解码；示例限定单线程父进程，多线程fork子进程在exec前的可调用函数另受异步信号安全约束。


---

## LINUX-Q08

**waitpid 的作用是什么？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q08)

**面试答案：**

waitpid 可以等待指定子进程的状态变化；常见用法是等待子进程退出并回收退出状态。返回状态需要用 `WIFEXITED/WEXITSTATUS` 或 `WIFSIGNALED/WTERMSIG` 等宏解析，不能把 status 原值直接当退出码。等待也可能被信号中断，因此应检查返回值。

**解析：**

等待和回收是相关但不同的动作。子进程已经结束时，可以直接回收，不需要再等它运行。


---

## LINUX-Q09

**僵尸进程是什么？kill 能不能解决？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q09)

**面试答案：**

通常，子进程已结束但退出状态尚未被回收，就处于僵尸状态。它不再执行用户代码，但内核还保留 PID、退出状态等信息，过多僵尸会占用进程管理资源。它已经退出，继续 kill 不能代替回收；应修复父进程的 wait/waitpid 等回收路径。

**解析：**

不要把僵尸当成“后台一直运行、杀不掉的线程”。特殊 SIGCHLD 配置会改变僵尸产生规则，暂不作为本轮主线。


---

## LINUX-Q10

**孤儿与僵尸有什么区别？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q10)

**面试答案：**

孤儿关注父进程先退出后子进程被重新收养的关系；僵尸关注子进程已经退出、退出状态尚未回收。孤儿并不必然是异常进程，僵尸也不意味着原父进程永远活着。Linux 下收养者可能是相应的 init 或 subreaper。

**解析：**

“父死子活”和“子死父活”可以帮助初学记忆，但不能替代完整定义。


---

## LINUX-Q11

**用户态、内核态与系统调用是什么关系？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q11)

**面试答案：**

用户态与内核态区分执行权限。用户程序通过系统调用进入内核请求受控服务，如文件操作和进程管理；内核检查参数和权限后执行，再返回用户态。普通用户函数调用通常不跨这个权限边界；root 身份也不等于程序一直在 CPU 内核态运行。

**解析：**

用户身份权限与 CPU 执行权限是两个层面。“用户程序调用驱动”通常是通过系统调用路径，不是普通跨程序函数调用。


---

## LINUX-K-001

**系统调用的完整流程是什么？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-k-001)

### 30~90 秒面试版
用户程序通常通过 libc 等封装调用系统调用接口，CPU 执行专用陷入指令后从用户态切换到内核态。内核根据系统调用号进入对应处理函数，检查参数和权限，执行 VFS、网络、驱动或内存等子系统逻辑；如果需要等待资源，当前线程可能阻塞并被调度出去。完成后内核设置返回值，恢复用户态上下文并返回用户程序。

### 追问
- 系统调用和普通函数调用差在哪？
- 用户态/内核态切换为什么有开销？
- `read()` 为什么可能阻塞？
- 驱动在链路中的位置是什么？


---

## LINUX-Q12

**一次系统调用一定导致进程或线程切换吗？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q12)

**面试答案：**

不一定。系统调用涉及用户态到内核态的模式切换，但可以仍在当前线程上下文中完成，然后返回原线程。若调用需要等待数据而阻塞，或出现其他调度条件，才可能切到别的线程。不能把模式切换与任务上下文切换混为一谈。

**解析：**

“执行谁的代码”与“当前在为哪个线程执行”不是同一个问题。进入内核执行代码，不等于内核总要另开一个线程接手。


---

## LINUX-Q13

**驱动为什么不能直接解引用用户传入的指针？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q13)

**面试答案：**

用户指针不能被默认信任，可能非法、权限不满足或触发缺页。内核通常使用 `copy_from_user/copy_to_user` 或单值 uaccess 接口完成受控访问，并检查结果。这些接口可能睡眠，因此还要符合上下文限制，不能随意放进 ISR 或持有自旋锁的区域。

**解析：**

这里不是说用户地址与内核永远没有映射关系，而是不能绕过内核规定的安全访问方式。`copy_*_user` 返回未复制的字节数，不是普通的 0/-1 约定。


---

## LINUX-VM-001

**虚拟地址与 swap**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-001)

### 面试回答

打印的是当前进程中的虚拟地址，不是这个变量在 RAM 中的物理地址。不开 swap 也能使用虚拟内存：地址转换、权限隔离和进程受控共享并不依赖必须有交换空间。

### 原因与解析

`value` 是数据，`&value` 是程序用来定位它的地址。在启用 MMU 的 Linux 中，这个地址还需要按当前地址空间映射。swap 只是可能的后备存储机制，不能把它当作“虚拟内存”的全部定义。

### 易错边界

看到一个地址，先说它属于哪个进程的虚拟地址空间；不要仅凭数字推断物理地址。 依据：[VM-S1](https://docs.kernel.org/admin-guide/mm/concepts.html)[VM-S2](https://docs.kernel.org/mm/page_tables.html)。


---

## LINUX-VM-002

**父子相同地址与私有数据**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-002)

### 面试回答

不矛盾。父子进程拥有各自的地址空间，普通私有变量在逻辑上独立；相同虚拟地址数值可以根据不同进程的映射对应不同物理页。fork 的 COW 还允许一开始暂时共享，写入时再按需分离。

### 原因与解析

例如父子都使用虚拟地址 V。子进程写入后，它的 V 可以改为指向自己的页面；父进程的 V 仍指向保有原值的页面。打印只观察到了 V，没有观察页表，更没有测出具体物理页号。因此“地址一样”不证明“此刻共享同一可写对象”。

### 易错边界

“逻辑独立”和“物理暂时共享”可以同时成立。不要把共享映射的行为套到普通私有变量。 依据：[VM-S1](https://docs.kernel.org/admin-guide/mm/concepts.html)[VM-S5](https://man7.org/linux/man-pages/man2/fork.2.html)。


---

## LINUX-VM-003

**页面粒度与对象大小**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-003)

### 面试回答

一个 4 字节 int 不必独占一整页。页是映射管理粒度，同页里可以放多个对象。4 MiB 在 4 KiB 基本页假设下包含 1024 个页面。

### 原因与解析

计算为 (4×1024×1024)/(4×1024)=1024。程序仍然按指令读写字节或整数，页的存在并不会让一次普通 int 读写变成必须整页拷贝。此处只计算页面数，不计算页表层级或分配器额外开销。

### 易错边界

4 KiB 是题设，不是所有平台恒定值；页面数与元素个数不要混用。 依据：[VM-S1](https://docs.kernel.org/admin-guide/mm/concepts.html)[VM-S15](https://man7.org/linux/man-pages/man3/sysconf.3.html)。


---

## LINUX-VM-004

**虚拟地址换算**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-004)

### 面试回答

页内偏移为 0xABC，物理页框起始地址为 0x7000，最终物理地址为 0x7ABC。

### 原因与解析

虚拟地址 0x2ABC 除以 0x1000 得页号 2，余数是 0xABC。根据题设，页号 2 映射到物理页框 7，所以新页的基地址为 7×0x1000=0x7000。再加原偏移，得到 0x7ABC。

```text
0x2ABC → 虚拟页号2 + 偏移0xABC
页表：虚拟页号2 → 物理页框7
0x7000 + 0xABC = 0x7ABC
```

### 易错边界

转换不改变页内偏移；偏移单位是字节，不是 int。 依据：[VM-S2](https://docs.kernel.org/mm/page_tables.html)。


---

## LINUX-VM-005

**虚拟连续不等于物理连续**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-005)

### 面试回答

不成立。malloc 返回的块可以在虚拟地址上连续，但跨越的不同虚拟页可以映射到分散的物理页框。普通下标连续访问由地址转换保持，不要求全部物理页连在一起。

### 原因与解析

假设两个连续虚拟页分别映射到物理页框9和3，程序跨页继续访问时使用下一页映射即可。不能由 p、p+4096 的地址数值连续推出物理页连续，更不能据此把 p 直接交给 DMA 硬件。

### 易错边界

本题不要求展开 DMA/IOMMU。只需明确用户指针、物理位置不是同一概念。 依据：[VM-S1](https://docs.kernel.org/admin-guide/mm/concepts.html)。


---

## LINUX-VM-006

**合法区域与已有映射**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-006)

### 面试回答

不能直接判非法。内核登记的合法区域及其权限，说明进程是否允许使用该地址；页表反映该页目前的实际映射。地址合法但页面尚未准备好时，内核可通过正常缺页处理补齐映射。

### 原因与解析

先区分“这个位置允许我使用”和“这页现在已经准备到哪一步”。内核检查范围和权限后，可能分配匿名页、处理文件页或COW；不能单看一个尚未建立的页表项，就跳过合法性和映射类型判断。

### 易错边界

VMA 不是物理数据，MMU 不是页表本身。 依据：[VM-S1](https://docs.kernel.org/admin-guide/mm/concepts.html)[VM-S2](https://docs.kernel.org/mm/page_tables.html)。


---

## LINUX-VM-007

**TLB失效与两种Cache**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-007)

### 面试回答

TLB 可能仍缓存“虚拟页1→物理页框9”，只修改页表后还可能继续使用旧转换。因此内核需要按架构要求处理相关 TLB 记录。TLB 命中也不保证数据 Cache 命中，因为二者缓存的不是同一种东西。

### 原因与解析

TLB 解决“去哪里”，数据 Cache 解决“所需内容是否已经在较近的缓存里”。失效 TLB 只是让旧转换不再被继续使用，不是把物理数据清零；处理范围也不必永远是整个 TLB。

### 易错边界

不要把 TLB、页表、文件页缓存和数据 Cache 都叫“保存数据的缓存”而失去区分。 依据：[VM-S1](https://docs.kernel.org/admin-guide/mm/concepts.html)[VM-S3](https://docs.kernel.org/core-api/cachetlb.html)。


---

## LINUX-VM-008

**TLB miss与缺页区分**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-008)

### 面试回答

A：在题设的硬件页表遍历模型下，查到有效映射即可继续，不能仅因TLB miss认定缺页。B：命中了转换，但写权限不满足，仍可能产生权限异常。C：合法匿名页尚未准备，可走正常缺页处理后继续。

### 原因与解析

A是“转换缓存没记录”；B是“访问权限不允许直接执行”；C是“合法访问仍需要准备页面”。其中B还需由内核判断：到底是可处理的COW，还是实际非法写入，不能省略这个判断。

### 易错边界

命中不等于无条件允许，未命中不等于一定异常。 依据：[VM-S2](https://docs.kernel.org/mm/page_tables.html)。


---

## LINUX-K-002

**缺页异常：现象、判断、恢复**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-k-002)

### 面试回答

缺页异常表示当前访存不能通过现有映射或权限直接完成，需要内核处理。它既可能是按需分配和COW的正常路径，也可能是非法访问。对合法且可满足的请求，内核准备页面或更新映射后让程序继续；真正非法访问则按类型报告错误。

### 原因与解析

新匿名区首次写：先检查区域合法且可写，再准备实际页和映射。

COW写：内核识别逻辑上允许的私有写入，必要时复制页，再为写入方建立合适映射。

非法访问：不能任意补成合法，常见结果是SIGSEGV；某些文件映射错误会产生SIGBUS。

所以“异常”不等于“程序Bug”，“缺页”也不等于“内存一定耗尽”。

### 易错边界

保留仓库原题号。未在本轮重新核验其企业面经来源；以上是学习用机制解析。 依据：[VM-S1](https://docs.kernel.org/admin-guide/mm/concepts.html)[VM-S2](https://docs.kernel.org/mm/page_tables.html)[VM-S8](https://man7.org/linux/man-pages/man2/mmap.2.html)。


---

## LINUX-VM-010

**Minor fault并非错误报警**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-010)

### 面试回答

不能证明程序出错。Minor fault 表示本次缺页处理不需要I/O，新匿名页准备和某些COW可出现这种情况；Major fault 才表示服务缺页需要I/O。首次使用缓冲区时计数增长，可以是正常内存管理现象。

### 原因与解析

要结合阶段观察增量：第一次写入的新页面多，可能增加正常缺页；后续复用若页面仍在，增量可能少。该统计不是“错误次数”，也不能单凭minor计数判断物理内存不足。

### 易错边界

Major不是“更严重的C代码Bug”，Minor也不是“可以忽略的Bug”。 依据：[VM-S6](https://man7.org/linux/man-pages/man2/getrusage.2.html)。


---

## LINUX-VM-011

**malloc成功的实际含义**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-011)

### 面试回答

两句话都过度绝对。malloc成功提供可使用的动态内存块，但不保证全部页面已驻留；同时，分配器可能复用已经驻留的页面，所以不能保证全部页面尚未分配。第一次访问是否缺页，取决于当时映射、权限和驻留情况。

### 原因与解析

可以按两条路径说明：

```text
新匿名区域：获得地址 → 第一次实际写页 → 可能缺页并准备物理页
已有空闲块：分配器复用 → 页面可能早已可用 → 不必因第一次使用就缺页
```

malloc是库接口，未必每次向内核申请新VMA；返回内容也没有初始化保证。

### 易错边界

“malloc成功≠所有物理页到位”是核心。“页表项此时一律无效”不是通用结论。 依据：[VM-S4](https://man7.org/linux/man-pages/man3/malloc.3.html)。


---

## LINUX-VM-012

**overcommit的适用边界**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-012)

### 面试回答

两句都不对。模式1允许宽松超额承诺，但不能消除地址空间、资源限制等所有失败因素；模式2按承诺额度严格检查，并不要求申请时立即把所有物理页都准备好。承诺核算与按需驻留是不同问题。

### 原因与解析

模式0是默认启发式策略，不保证任意超大申请成功。模式1不是malloc永不失败。模式2控制系统commit额度，不等于禁止所有虚拟空间超过RAM，也不等于关闭按需分页。后续真实内存压力无法解决时可能发生OOM，但本题不要求人为触发。

### 易错边界

这些是对前面提供的讲解范例的边界核对，不是用户已经答错的记录。 依据：[VM-S4](https://man7.org/linux/man-pages/man3/malloc.3.html)[VM-S7](https://docs.kernel.org/mm/overcommit-accounting.html)。


---

## LINUX-VM-013

**PRIVATE、SHARED与原文件**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-013)

### 面试回答

PRIVATE阶段，自己的映射首字节是X，原文件首字节仍是A。解除后重新建立SHARED映射，把首字节改为Y并成功同步，原文件首字节也可读到Y。两者都没有要求映射建立时立即复制或读取整个文件。

### 原因与解析

PRIVATE的写入是私有修改，不回写原文件；SHARED允许共享和更新后备文件。题设无并发写入者，所以可清楚观察这一差别。实际共享访问仍需同步，写回和断电持久性也不可混为一谈。

原材料引用的mmap_modes.c未随本次资料提供。本题按映射语义解析，不声称存在已验证的完整程序。实验需使用独立临时文件，先ftruncate到所需长度，再映射并检查MAP_FAILED/msync/munmap。

### 易错边界

mmap失败检查MAP_FAILED，不是NULL；解除用munmap，不是free。 依据：[VM-S8](https://man7.org/linux/man-pages/man2/mmap.2.html)[VM-S9](https://sourceware.org/glibc/manual/latest/html_node/Memory_002dmapped-I_002fO.html)。


---

## LINUX-VM-014

**VSZ、RSS与独占内存**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-014)

### 面试回答

都不能。2 GiB是虚拟空间规模，不表示这些范围都已使用2 GiB RAM。VmRSS描述当前驻留页面，包含可能共享的页面，且该接口统计可存在近似，因此也不等于精确独占120 MiB物理内存。

### 原因与解析

先看VmSize与VmRSS分别在量什么，再结合匿名、文件或共享映射理解用途。详细检查可考虑smaps/smaps_rollup。只看某一个数字就断言泄漏或独占量，证据不足。

### 易错边界

VSZ不是malloc总和；RSS不是不与其他进程共享的页面总和。 依据：[VM-S10](https://man7.org/linux/man-pages/man5/proc_pid_status.5.html)[VM-S11](https://man7.org/linux/man-pages/man5/proc_pid_statm.5.html)。


---

## LINUX-VM-015

**free与RSS下降**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-015)

### 面试回答

都不能直接得出。free把块交回分配器后，分配器可能保留空间供下次复用，所以RSS暂时不降不等于泄漏。另一方面，某条路径存在free不保证所有异常路径、所有分配块都被释放。

### 原因与解析

排查先固定工作负载，观察多轮后是否继续无界增长；再检查对象拥有者、申请释放是否匹配，以及合理缓存/分配器保留。内存检查工具和分配日志能提供更直接的证据，而不是只盯RSS。

### 易错边界

即使某次大块实验free后RSS下降，不能拿该单次结果反推所有小块也必须立即返还。 依据：[VM-S13](https://sourceware.org/glibc/manual/latest/html_node/Freeing-after-Malloc.html)。


---

## LINUX-VM-016

**实验为什么可能测错**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-016)

### 面试回答

system启动的cat读取/proc/self时，self指cat而不是父测试程序。另一个问题是实验必须检查malloc是否成功，并保证实际写入没有被编译器当无用操作删除，否则观察不到预期变化不能证明按需分配不存在。

### 原因与解析

改为程序自身fopen读取/proc/self/status，或外部指定测试程序PID。statm单位是页数，不能直接按字节读。记录运行阶段，控制请求大小，比较VmSize、RSS和缺页增量。数值差异还要考虑分配器复用、已驻留页、页大小和统计精度。

可通过有限大小缓冲区上的volatile字节写入保持实验操作；它不是线程安全手段。原材料所述程序没有随包提供，以上是设计说明，不是本次实测。

### 易错边界

不要用“cat的RSS没变”证明测试进程没分配物理内存；不要把未核验的预期输出当实测。 依据：[VM-S11](https://man7.org/linux/man-pages/man5/proc_pid_statm.5.html)[VM-S12](https://man7.org/linux/man-pages/man5/proc_self.5.html)。


---

## LINUX-VM-017

**首帧延迟的内存排查**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-vm-017)

### 面试回答

malloc提前完成，不保证首次真正使用缓冲区时没有缺页和页面准备开销。可以比较首次/稳态时延和缺页增量，再验证预触页、缓冲区复用是否改善。mlock等只解决部分驻留问题，不保证调度、I/O、同步和计算的所有截止时间。

### 原因与解析

先把缺页当候选原因，而不是直接当根因。固定输入，记录各阶段耗时，观察minor/major变化；初始化阶段实际访问页面后再对比。必要时评估锁页权限与额度并检查失败。新分配、fork/COW或其他系统活动也不能因用过一次mlock就不分析。

### 易错边界

这是基于本次学习机制的工程分析题，不是对你当前成像或RK3588代码的实测诊断。 依据：[VM-S14](https://man7.org/linux/man-pages/man2/mlock.2.html)。


---

## LINUX-IO-001

**fd 返回 0 是不是打开失败？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-io-001)

### 面试答案

文件描述符是进程描述符表中的非负整数索引。open 成功可以返回 0，失败才返回 -1。0 常用于标准输入，但标准输入关闭后，编号 0 可以被后续 open 复用。fd 不是指针，不能把判断空指针的习惯直接用于 fd。[IO-S1](https://man7.org/linux/man-pages/man2/open.2.html)

### 逐项解析

题目中 0 是最小可用编号，因此返回 0 正常。`if (fd == 0)` 会把成功误判为失败，还可能漏掉真正的 -1。正确片段：

```c
int fd = open("data.txt", O_RDONLY);
if (fd == -1) {
    perror("open");
    /* 此分支处理打开失败 */
}
```

不同进程的 fd=3 不必对应同一对象；查表时有“当前进程的描述符表”这个上下文。

---

## LINUX-IO-002

**分别 open、dup、整数赋值的区别。**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-io-002)

### 面试答案

文件偏移保存在内核打开文件描述对象里。分别 open 同一个普通文件，会得到不同打开对象，所以偏移各自独立。dup 创建一个新描述符入口，但指向原打开对象，偏移共享。`fd2=fd1` 则只复制整数，根本没有创建新描述符。[IO-S1](https://man7.org/linux/man-pages/man2/open.2.html)[IO-S4](https://man7.org/linux/man-pages/man2/dup.2.html)

### 逐项解析

假设文件 ABCDEFGHIJ、顺序执行、每次实际读到 3 字节：

| 场景 | fd1 第一次读 | fd2 随后读 | 关闭 fd1 后的 fd2 |
|---|---|---|---|
| 分别 open | ABC | ABC | 独立入口仍可用 |
| dup | ABC | DEF | 独立入口仍可用，仍指向原打开对象 |
| fd2=fd1 | ABC | DEF | 没有另一个入口；旧整数不能继续作为原对象使用 |

后二者读取结果相同，不代表资源管理方式相同。dup 的独立入口可单独 close；整数别名不能靠“关两次”释放两份资源。[IO-S5](https://man7.org/linux/man-pages/man2/close.2.html)

**易错边界**：共享偏移不等于每个线程/进程会自动获得一帧完整数据。无同步并发读取时，谁先读取、每次读取多少仍影响应用行为。

---

## LINUX-IO-003

**写 10 字节却返回 4，如何继续？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-io-003)

### 面试答案

write 返回值才是实际完成量。返回 4 后，应从原缓冲区偏移 4 的位置开始，请求写剩下 6 字节，并继续检查结果。不能从原地址重新写 10 字节，否则可能重复前 4 字节。后续失败不会自动回滚已经写入的内容。[IO-S3](https://man7.org/linux/man-pages/man2/write.2.html)

### 逐项解析

```text
data: A B C D E F G H I J
      已完成   待完成
下一次：write(fd, data + 4, 6)
```

文件位置由内核推进，缓冲区位置由应用推进。二者不是同一个“指针”。

### 完整辅助函数：阻塞普通文件写入

```c
#include <errno.h>
#include <stddef.h>
#include <unistd.h>

/*
 * 返回0：全部字节已被write接受。
 * 返回-1：失败，errno说明原因；此前可能已写入部分字节。
 * 前提：阻塞式普通文件、buffer覆盖length字节。
 * 单次请求上限用于避免极大length直接超出一次write的限制。
 * 不提供事务回滚、持久化保证或非阻塞事件循环。
 */
int write_all_blocking(int fd, const void *buffer, size_t length)
{
    if (buffer == NULL && length != 0) {
        errno = EINVAL;
        return -1;
    }

    const unsigned char *p = buffer;
    size_t done = 0;

    while (done < length) {
        size_t request = length - done;
        if (request > 1024u * 1024u)
            request = 1024u * 1024u;

        ssize_t n = write(fd, p + done, request);
        if (n > 0) {
            done += (size_t)n;
        } else if (n == -1 && errno == EINTR) {
            continue;
        } else {
            if (n == 0)
                errno = EIO;
            return -1;
        }
    }
    return 0;
}
```

**解析**：只在返回正数时累计；EINTR 在本例中重试；0 表示无进展，不能永远循环。非阻塞 EAGAIN 应留给事件循环保存进度并等待，不应该照搬这个函数无限重试。[IO-S3](https://man7.org/linux/man-pages/man2/write.2.html)

**官方补充**：若改用 pipe/Socket，还要设计 SIGPIPE/EPIPE、超时和取消策略。函数写完不等于 fsync 完成。

---

## LINUX-Q14

**例题：短读、短写与共享偏移。**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q14)

### 面试答案

fork 继承的描述符引用同一个内核打开对象，父子共享文件偏移。子进程实际读完 10 字节后，若初始偏移 0 且没有其他访问，父从偏移 10 继续。父 close 只删除父自己的入口，不删除子进程的描述符。TCP 一次读到 3 字节不是一帧完成，应保留已有数据并按协议继续接收，处理 EOF、EINTR 和 EAGAIN。[IO-G3](https://github.com/3137959601/-/blob/main/答案/Linux答案.md)[IO-S6](https://man7.org/linux/man-pages/man2/fork.2.html)[IO-S12](https://man7.org/linux/man-pages/man7/tcp.7.html)

### 逐项解析

**第一问**：不是父进程“还没读过所以从头读”。书签属于共享打开记录。

**第二问**：父子有各自的描述符表。共享底层对象不等于关闭其中一项会删除另一项。

**第三问**：若协议明确一帧固定 20 字节，第一次保存 3 字节，下次放到 `buf + 3`，最多读剩余 17 字节；再次短读则继续累计。变长协议先解析头部/长度，不能把随意申请的缓冲区长度当消息长度。

```c
/* 原理片段；EOF/错误/超时应由外层处理。 */
ssize_t n = read(fd, buf + received, expected - received);
if (n > 0)
    received += (size_t)n;
```

**Debug 检查**：记录每次实际 n、累计 received、帧序号和长度字段。不要用 strlen 统计二进制协议，也不要在短读时把已收字节清空。

---

## LINUX-IO-004

**EAGAIN 是断线吗？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-io-004)

### 面试答案

非阻塞 TCP read 返回 -1/EAGAIN 或 EWOULDBLOCK，通常表示当前没有可读数据，不等于连接失败。程序应保留解析状态，等待可读事件后再读，不能立即无间隔重复造成忙等。请求长度大于 0 时，接收方向正常结束且数据已读尽，read 返回 0，这是 EOF，与暂时无数据不同。[IO-G2](https://github.com/3137959601/-/blob/main/嵌入式秋招八股文_Linux.md)[IO-S2](https://man7.org/linux/man-pages/man2/read.2.html)

### 逐项解析

当前请求如果不能完成任何读取：非阻塞返回，而不是替应用在后台等待并填充 buf。之后要读取，必须再调用接口。

```text
n > 0                 → 消费本次字节，累计进度
n == 0                → 按接收方向结束处理
n == -1且EAGAIN       → 等待就绪
n == -1且EINTR        → 本例按业务允许重试
其他错误              → 记录/关闭/重连等，视业务决定
```

一个专用接收线程可以合理采用阻塞 read；非阻塞不天然更快。需要同一线程管理多个 fd 时，常与 poll/epoll 结合。

**Debug 线索**：CPU 很高但几乎没有接收进度，若调用日志持续出现 EAGAIN，先检查是否把非阻塞写成了忙轮询，而不是先怀疑内核坏了。这是排查建议，不是对某个实际工程的结论。

---

## LINUX-PIPE-001

**pipe 后 fork，分别关闭哪一端？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-pipe-001)

### 面试答案

pipe 在内核建立单向字节流通道，通过数组返回读端和写端两个描述符。fork 后父子都继承两端。父写子读时，父立即关闭不用的读端，子立即关闭不用的写端。发送完成后父关闭写端，让子读完剩余字节后得到 EOF，随后父回收子进程。[IO-S7](https://man7.org/linux/man-pages/man2/pipe.2.html)

### 逐项解析

`pipe()` 返回 0 表示成功，`pipefd[0]` 则是一个具体读端 fd，例如 3。两者不是同一个返回信息。

```text
父：关闭 pipefd[0]；向 pipefd[1] 写；完成后 close(pipefd[1])。
子：关闭 pipefd[1]；从 pipefd[0] 读；EOF 后 close(pipefd[0])。
```

原材料提到pipe_parent_child.c，但本次资料未提供该程序，不能声称已编译验证。上面的端口关闭顺序是核心片段；自行实现时还应检查fork失败、短读短写、EINTR、SIGPIPE/EPIPE和waitpid回收。

---

## LINUX-PIPE-002

**子进程读完 HELLO，却等不到 EOF。**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-pipe-002)

### 面试答案

EOF 的条件是管道数据已读完，并且所有写端引用都关闭。父进程 close 只能关闭父自己的写端；子进程如果还留着继承的写端，内核就认为仍有发送方存在。因此子下一次阻塞 read 会等待。修复是在 fork 后立即关闭子不用的写端，检查是否还有 dup 或其他进程继承留下的写端。[IO-G2](https://github.com/3137959601/-/blob/main/嵌入式秋招八股文_Linux.md)[IO-S8](https://man7.org/linux/man-pages/man7/pipe.7.html)

### 原因与排查

“我不会用它写”是程序员的意图，不是内核能判断的事实。内核依据仍存活的描述符引用判断。

建议按顺序查：卡在哪个调用 → 哪些进程仍持有管道端 → fork/dup 后每条分支是否关闭无用端 → 错误退出路径是否也正确清理。改成非阻塞只可能把等待变成 EAGAIN，并没有修复 EOF 条件。

---

## LINUX-PIPE-003

**为什么先 waitpid 再关闭写端会卡住？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-pipe-003)

### 面试答案

子进程等 EOF 才退出，而 EOF 需要父关闭最后的写端。父又先 waitpid 等子退出，再去关闭写端，就形成了等待环。修复顺序是发送完后先 close 写端，再 waitpid；子读完所有数据并见到 EOF 后退出。[IO-S7](https://man7.org/linux/man-pages/man2/pipe.2.html)[IO-S8](https://man7.org/linux/man-pages/man7/pipe.7.html)

### 等待关系

```text
父等子退出 ──→ 子等EOF ──→ EOF等父close ──→ 父仍在wait
```

单纯提高优先级、加 sleep、增大管道容量都不解除这个等待环。题目已限制数据量小于可用容量，是为了排除“父还没写完就因管道满而等待”的另一类场景。

---

## LINUX-FIFO-001

**FIFO 与 pipe、普通磁盘文件有什么不同？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-fifo-001)

### 面试答案

FIFO 是文件系统里有名字的管道。独立启动且权限允许的进程可以通过同一路径分别 open 读端和写端，不必靠父子继承找到通道。路径只是通信入口，数据由内核管道缓冲区传递，不作为普通文件内容持久保存。它和 pipe 一样是字节流，不自动提供业务消息边界。[IO-S9](https://man7.org/linux/man-pages/man7/fifo.7.html)[IO-S10](https://man7.org/linux/man-pages/man3/mkfifo.3.html)

### 逐项解析

**为什么有名字**：进程不需要拿到别人进程内的 fd 数字，只需要访问相同 FIFO 节点。

**数据存哪里**：传输的字节暂存于内核对象。文件系统有路径节点，不意味着数据写进该节点对应的“普通文件正文”。

**为何进程退出路径仍在**：close 解除描述符，删除名字是另外的 unlink/rm 操作。关闭所有端后，不要指望重开 FIFO 能取回之前未消费的数据。[IO-S9](https://man7.org/linux/man-pages/man7/fifo.7.html)

**选择边界**：本机简单流式进程通信可考虑 FIFO；要重启后保留数据，应考虑普通文件或其他持久化设计；要每个接收者都收到相同数据，不能把多个读者共用一根 FIFO 当广播。

---

## LINUX-FIFO-002

**为什么 FIFO 程序卡在 open？**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-fifo-002)

### 面试答案

FIFO 默认阻塞打开时，单独打开写端会等待读端出现；单独打开读端也会等待写端出现。因此建立节点不等于两端已经连接。写端使用 O_NONBLOCK 且没有读者时，open 返回 -1、errno=ENXIO，而不是返回一个之后慢慢等待的写端 fd。[IO-S9](https://man7.org/linux/man-pages/man7/fifo.7.html)[IO-S10](https://man7.org/linux/man-pages/man3/mkfifo.3.html)

### 逐项解析

区分两个阶段：

```text
open：等待另一端打开。
read/write：端口已打开后，等待数据或缓冲空间。
```

无读者时写端非阻塞 open 的 ENXIO，与已经打开的非阻塞读端暂时无数据的 EAGAIN，是不同场景。

读端非阻塞 open 可以在无写者时成功；此时如果缓冲为空、无写者，read 可以返回 0。FIFO 节点还可被新的写者打开，所以服务程序需要明确“收到一次 EOF 后退出还是重新等待”的生命周期策略。

**不要用错误方法掩盖问题**：Linux 允许 O_RDWR 打开 FIFO，但这样进程自己也持有写端，可能改变 EOF 判定。这里不推荐把它当“万能解决阻塞”的开关。[IO-S9](https://man7.org/linux/man-pages/man7/fifo.7.html)


---

## LINUX-SYNC-001

**共享内存创建、大小、映射与清理**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-sync-001)

### 面试级回答

shm_open找到对象并给fd，ftruncate设置存储范围，mmap才把它映射到本进程并返回访问地址。新对象长度为0，不能仅因为有fd就访问1 MiB。清理时close关fd，munmap撤销本进程映射，shm_unlink删名字；需要协调其他使用者的生命周期。

### 原因与解析

先由创建者设置大小与初始化，再通过约定允许B使用；B也要检查对象大小、映射结果和权限。双方各得到自己的映射地址。不要由所有打开者重复初始化同步对象。以上是接口分工，不是一个已实现的跨进程启动协议。

资料依据：[SYNC-S1](https://man7.org/linux/man-pages/man7/shm_overview.7.html)，见[知识正文参考资料](../嵌入式秋招八股文_Linux.md#参考资料)。


---

## LINUX-Q15

**共享内存并不自动同步**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q15)

### 面试级回答

不可仅靠volatile索引和私有malloc指针完成可靠共享。volatile不建立互斥或跨执行者的数据发布协议；共享区保存的malloc地址只在创建它的进程地址空间中有意义。应把载荷放在共享区，使用内嵌数组或偏移/索引，并配合进程共享同步对象。

### 原因与解析

具体修复按四步：①载荷与描述信息共同放入共享区域；②用槽位索引标识资源，而非私有地址；③约定空闲→写入→就绪→读取→归还；④检查长度和越界，处理满缓冲、异常退出。一个信号量能告诉你空闲数，却不会自动告诉你哪一槽空闲。该题是仓库已有题，本次没有用户新作答，不登记个人错误。

资料依据：[SYNC-R1](https://github.com/3137959601/-/blob/main/嵌入式秋招八股文_Linux.md), [SYNC-S1](https://man7.org/linux/man-pages/man7/shm_overview.7.html), [SYNC-S4](https://man7.org/linux/man-pages/man3/sem_init.3.html), [SYNC-S6](https://man7.org/linux/man-pages/man3/pthread_mutexattr_getpshared.3.html)，见[知识正文参考资料](../嵌入式秋招八股文_Linux.md#参考资料)。


---

## LINUX-SYNC-002

**计数可以保留，条件变量通知不能当计数用**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-sync-002)

### 面试级回答

信号量post后计数从0到1，若没有别人消费，之后wait可取得这次机会并减为0。条件变量signal在无人等待时不保存将来的通知次数。消费者必须检查共享谓词，而不是依赖是否听到过signal。

### 原因与解析

区别不是“一个能通知，一个不能通知”，而是状态保存在哪里。信号量自带计数；条件变量方案的业务状态在count、full或队列里。即使无人接到signal，已入队的数据也还在，后来消费者加锁检查到非空就不必等。

资料依据：[SYNC-S2](https://man7.org/linux/man-pages/man7/sem_overview.7.html), [SYNC-S3](https://man7.org/linux/man-pages/man3/sem_wait.3.html), [SYNC-S9](https://man7.org/linux/man-pages/man3/pthread_cond_broadcast.3p.html)，见[知识正文参考资料](../嵌入式秋招八股文_Linux.md#参考资料)。


---

## LINUX-SYNC-003

**pshared不会自动创建共享对象**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-sync-003)

### 面试级回答

不能。两个私有全局sem_t是两份不同对象，名字相同、pshared都为1也不能合并它们。无名跨进程信号量必须放在双方实际映射到的同一共享区域，再用非零pshared初始化。

### 原因与解析

先建立共享内存，把sem_t放进去；只有创建者初始化，随后其他进程才能使用。另一种接口路线是双方通过同名sem_open找到有名信号量。本阶段已讲无名共享对象方案，有名接口只作为边界补充，不要求同时背两套完整代码。

资料依据：[SYNC-S2](https://man7.org/linux/man-pages/man7/sem_overview.7.html), [SYNC-S4](https://man7.org/linux/man-pages/man3/sem_init.3.html)，见[知识正文参考资料](../嵌入式秋招八股文_Linux.md#参考资料)。


---

## LINUX-SYNC-004

**共享槽位的归还时机**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-sync-004)

### 面试级回答

存在覆盖风险。p只是原缓冲区的地址副本，post(empty)却已经允许生产者改写那块内存。消费者随后处理时，可能与新帧写入冲突。应在使用结束后归还，或者在交还前把所需数据复制到独立缓冲区。

### 原因与解析

修复一：wait(ready)→process共享数据并确保不再保留引用→post(empty)。修复二：wait(ready)→完整复制数据→post(empty)→处理副本。若要避免全量复制又想并行，使用多个槽位和所有权队列，让生产者写另一个空闲槽位。这里只给设计方向，不把多槽位协议当作已完成代码。

资料依据：[SYNC-R1](https://github.com/3137959601/-/blob/main/嵌入式秋招八股文_Linux.md), [SYNC-S2](https://man7.org/linux/man-pages/man7/sem_overview.7.html), [SYNC-S3](https://man7.org/linux/man-pages/man3/sem_wait.3.html)，见[知识正文参考资料](../嵌入式秋招八股文_Linux.md#参考资料)。


---

## LINUX-SYNC-005

**写方加锁不能保护不遵守协议的读方**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-sync-005)

### 面试级回答

不能保证。Mutex约束的是参与同一加锁协议的访问者，不会自动阻止未加锁的普通内存读取。读方也要用同一Mutex，在锁内取得完整快照，然后解锁并在锁外使用独立副本。

### 原因与解析

读方流程：lock→snapshot=config→unlock→process(snapshot)。若结构体里还有指向可变共享载荷的指针，仅复制结构体未必获得完整独立快照；需要继续约定指针指向对象的生命周期与一致性。简单两个int的教学例子不存在这层额外所有权。

资料依据：[SYNC-R1](https://github.com/3137959601/-/blob/main/嵌入式秋招八股文_Linux.md), [SYNC-S5](https://man7.org/linux/man-pages/man3/pthread_mutex_lock.3p.html)，见[知识正文参考资料](../嵌入式秋招八股文_Linux.md#参考资料)。


---

## LINUX-SYNC-006

**通知丢失不等于数据丢失**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-sync-006)

### 面试级回答

标准写法不会因为消费者晚到就必然睡下。它先拿锁检查count，看到count=1就跳过while中的wait并消费。条件变量不保存通知，但队列保存数据，谓词保存可继续的判断依据。

### 原因与解析

危险的是无条件wait，或自己把“解锁”和“建立等待”拆开。正确消费者先持锁检查，不满足才cond_wait；生产者在同一Mutex下改状态并通知。先通知后等待本身不是一定出Bug，必须看有没有检查真实状态。

资料依据：[SYNC-S8](https://man7.org/linux/man-pages/man3/pthread_cond_wait.3p.html), [SYNC-S9](https://man7.org/linux/man-pages/man3/pthread_cond_broadcast.3p.html)，见[知识正文参考资料](../嵌入式秋招八股文_Linux.md#参考资料)。


---

## LINUX-SYNC-007

**线程创建返回码与参数生命周期**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-sync-007)

### 面试级回答

返回0代表创建成功，线程标识通过tid输出。第四个参数传的是指针值，并不自动复制value。若value已初始化、创建后不再修改，并且main在join成功前不结束其作用域，worker通过该指针读取是合理的。

### 原因与解析

常见错误：把rc当线程ID；用perror直接解释pthread的非零返回；循环把同一&i传给所有线程；函数返回后线程继续访问局部参数。参数为结构体也一样：复制地址不是复制结构体，地址有效还要满足并发读写同步。完整可运行示例见本题下方程序。

资料依据：[SYNC-S10](https://man7.org/linux/man-pages/man3/pthread_create.3.html), [SYNC-S11](https://man7.org/linux/man-pages/man3/pthread_join.3.html)，见[知识正文参考资料](../嵌入式秋招八股文_Linux.md#参考资料)。


### 完整程序：pthread_basic.c

从本次压缩包收录。将代码单独保存为pthread_basic.c，在Linux执行：

```bash
cc -std=c11 -O2 -Wall -Wextra -Wpedantic -pthread pthread_basic.c -o demo
./demo
```

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* pthread 入口：接收一个指针，并返回一个指针。 */
static void *worker(void *arg)
{
    const int *value = arg;
    printf("worker: value = %d\n", *value);
    return NULL;  /* 结束当前工作线程，不结束整个进程。 */
}

int main(void)
{
    pthread_t tid;
    int value = 100;

    /* value 在创建前初始化；创建后不再修改，直到 join 完成。 */
    int rc = pthread_create(&tid, NULL, worker, &value);
    if (rc != 0) {
        fprintf(stderr, "pthread_create: %s\n", strerror(rc));
        return EXIT_FAILURE;
    }

    /* 主线程在这里等待，value 的生命周期覆盖 worker 的全部访问。 */
    rc = pthread_join(tid, NULL);
    if (rc != 0) {
        fprintf(stderr, "pthread_join: %s\n", strerror(rc));
        return EXIT_FAILURE;
    }

    puts("main: worker finished");
    return EXIT_SUCCESS;
}
```

value在创建前初始化，之后不修改，join保证作用域覆盖worker的全部访问。压缩包附带的历史验证记录保存在仓库外备份中；本次整理未在Linux或开发板重新运行，不能把历史记录当成本次测试。

---

## LINUX-SYNC-008

**先创建协作者，再等待完成**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-sync-008)

### 面试级回答

会在特定执行路径下卡住：生产者填满单槽位后等消费者；消费者尚未创建；main又在join等生产者。应先完成两者create，再join两者，最后销毁它们使用的共享对象和同步原语。

### 原因与解析

正确动作：初始化channel→create消费者和生产者→join双方→destroy cond/mutex。这里只需要保证双方都已创建，不要求固定哪个先create。测试成功不证明覆盖了所有调度，只验证本次正常运行路径。

资料依据：[SYNC-S10](https://man7.org/linux/man-pages/man3/pthread_create.3.html), [SYNC-S11](https://man7.org/linux/man-pages/man3/pthread_join.3.html)，见[知识正文参考资料](../嵌入式秋招八股文_Linux.md#参考资料)。


### 完整程序：producer_consumer.c

从本次压缩包收录。将代码单独保存为producer_consumer.c，在Linux执行：

```bash
cc -std=c11 -O2 -Wall -Wextra -Wpedantic -pthread producer_consumer.c -o demo
./demo
```

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* 教学模型：同一进程、一个生产者、一个消费者、一个 int 槽位。 */
enum { ITEM_COUNT = 5 };

typedef struct {
    pthread_mutex_t mutex;
    pthread_cond_t not_empty;
    pthread_cond_t not_full;
    int value;
    int full;   /* 0：槽位空；1：有尚未取走的数据。必须在 mutex 下访问。 */
} Channel;

/*
 * 教学错误策略：pthread API 失败时报告操作名和错误码，终止整个示例。
 * 不是可恢复错误处理框架；真实产品通常还需 stop 状态与统一清理流程。
 */
static void check(int rc, const char *operation)
{
    if (rc != 0) {
        fprintf(stderr, "%s: %s (code=%d)\n", operation, strerror(rc), rc);
        exit(EXIT_FAILURE);
    }
}

static void *producer(void *arg)
{
    Channel *channel = arg;

    for (int value = 1; value <= ITEM_COUNT; ++value) {
        check(pthread_mutex_lock(&channel->mutex), "producer lock");

        /* 槽位尚有未消费的数据：释放锁并等待，醒后重新检查。 */
        while (channel->full) {
            check(pthread_cond_wait(&channel->not_full, &channel->mutex),
                  "producer wait not_full");
        }

        channel->value = value;
        channel->full = 1;
        check(pthread_cond_signal(&channel->not_empty), "signal not_empty");
        check(pthread_mutex_unlock(&channel->mutex), "producer unlock");
    }
    return NULL;
}

static void *consumer(void *arg)
{
    Channel *channel = arg;

    for (int i = 0; i < ITEM_COUNT; ++i) {
        check(pthread_mutex_lock(&channel->mutex), "consumer lock");

        /* 槽位空：释放锁并等待，醒后重新检查。 */
        while (!channel->full) {
            check(pthread_cond_wait(&channel->not_empty, &channel->mutex),
                  "consumer wait not_empty");
        }

        int value = channel->value;  /* 先复制到自己的局部变量。 */
        channel->full = 0;
        check(pthread_cond_signal(&channel->not_full), "signal not_full");
        check(pthread_mutex_unlock(&channel->mutex), "consumer unlock");

        /* 锁外只用自己的 value，不继续访问已交还的共享槽位。 */
        printf("consume: %d\n", value);
    }
    return NULL;
}

int main(void)
{
    Channel channel = {
        .mutex = PTHREAD_MUTEX_INITIALIZER,
        .not_empty = PTHREAD_COND_INITIALIZER,
        .not_full = PTHREAD_COND_INITIALIZER,
        .value = 0,
        .full = 0
    };
    pthread_t producer_tid;
    pthread_t consumer_tid;

    /* 同一个 &channel 交给两个线程，不复制 Channel。 */
    check(pthread_create(&consumer_tid, NULL, consumer, &channel),
          "create consumer");
    check(pthread_create(&producer_tid, NULL, producer, &channel),
          "create producer");

    /* 两个线程均已创建；join 只让 main 等待，不会停止另一个线程。 */
    check(pthread_join(producer_tid, NULL), "join producer");
    check(pthread_join(consumer_tid, NULL), "join consumer");

    /* 两个线程结束后，才销毁它们用过的同步对象。 */
    check(pthread_cond_destroy(&channel.not_empty), "destroy not_empty");
    check(pthread_cond_destroy(&channel.not_full), "destroy not_full");
    check(pthread_mutex_destroy(&channel.mutex), "destroy mutex");

    puts("main: both threads finished");
    return EXIT_SUCCESS;
}
```

两个线程先全部创建，再依次join；固定传输5个int，锁外仅使用局部副本。程序不实现无限生产、外部停止或故障恢复，pthread失败采用报告后exit的教学策略。压缩包附带的历史验证记录保存在仓库外备份中；本次整理未在Linux或开发板重新运行，不能把历史记录当成本次测试。

---

## LINUX-Q16

**条件变量复查与反向锁顺序**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q16)

### 面试级回答

唤醒后不能无条件消费：可能虚假唤醒，也可能别的消费者已经取走数据，必须重新持锁并用while检查谓词。反序拿两把锁时，要画出谁持有哪些锁、又等待谁；发现A持m1等m2、B持m2等m1的环，应统一获取顺序。

### 原因与解析

条件变量部分：wait返回时已重获传入的Mutex，但只保证可重新检查，不保证数据预留给自己。死锁部分：先看全部线程调用栈，再结合锁地址和获得/释放记录确认持有关系；不能只看到两个lock栈就认定互锁。修复优先一致锁顺序和缩短持锁范围。超时/trylock失败需要释放已取得的资源，不能跳过错误继续。保留原题编号，不重复制造用户错题。

资料依据：[SYNC-R1](https://github.com/3137959601/-/blob/main/嵌入式秋招八股文_Linux.md), [SYNC-S5](https://man7.org/linux/man-pages/man3/pthread_mutex_lock.3p.html), [SYNC-S8](https://man7.org/linux/man-pages/man3/pthread_cond_wait.3p.html), [SYNC-S13](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Threads.html)，见[知识正文参考资料](../嵌入式秋招八股文_Linux.md#参考资料)。


### 条件变量子问

生产者更新ready并通知后，消费者仍须在同一Mutex保护下while检查谓词。虚假唤醒或其他消费者先取走数据，都可令条件再次不成立；通知不等于资源已直接交给该线程。只有谓词成立，才可执行消费。

---

## LINUX-SYNC-010

**持锁join的循环等待**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-sync-010)

### 面试级回答

main持有worker检查stop所需的锁，又等待worker退出；worker无法取得锁，就无法完成退出；main的unlock放在join之后，导致双方互相等。join不会像cond_wait一样替你释放Mutex。

### 原因与解析

动作顺序：在锁内更新stop与必要共享状态→按协议signal/broadcast唤醒等待者→解锁→join→销毁资源。这里只给通用顺序，worker也必须把stop纳入等待谓词，并约定剩余数据是处理完还是丢弃；仅设置标志不保证睡眠线程醒来。安全退出将作为后续独立知识点。

资料依据：[SYNC-S8](https://man7.org/linux/man-pages/man3/pthread_cond_wait.3p.html), [SYNC-S11](https://man7.org/linux/man-pages/man3/pthread_join.3.html)，见[知识正文参考资料](../嵌入式秋招八股文_Linux.md#参考资料)。

---

## LINUX-SYNC-009

**futex等待只是线索**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-sync-009)

### 面试级回答

不能仅凭低CPU和futex等待断言死锁。正常Mutex竞争、条件等待或join也可能进入相似底层等待。需要查看所有相关线程的栈和业务状态，找出等待什么、谁能解除、那个执行者为何无法继续。

### 原因与解析

排查顺序：top -H看线程概况→GDB查看全部栈→定位业务帧、目标锁/条件→结合锁日志或状态确认持有者→形成依赖图。正常等外部数据不一定有环；真正A等B、B等A才是本例死锁。strace的futex仅提供底层线索。GDB附加会影响时序和实时性，测试环境取证后detach。

资料依据：[SYNC-S13](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Threads.html), [SYNC-S14](https://man7.org/linux/man-pages/man2/futex.2.html), [SYNC-S15](https://man7.org/linux/man-pages/man1/strace.1.html), [SYNC-S16](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Attach.html)，见[知识正文参考资料](../嵌入式秋招八股文_Linux.md#参考资料)。


---

## LINUX-EVENT-001

**读写锁：可以一起读，修改时必须独占**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-event-001)

**面试回答：**读写锁允许多个读者并发持锁，写者必须独占，因此可以用于具有并发读取需求、修改较少的共享数据。所有访问仍须遵守同一把锁，持读锁时不能修改共享状态。是否比 Mutex 快，要结合临界区长度、锁开销和写者等待情况实测，不能只看读写比例。

**解析：**读者不会修改同一份配置，能够共享读取；写者若与读者并行，读者可能观察到增益已更新而曝光尚未更新的混合状态。对极短读取，管理读写锁的成本可能超过并发收益。等待中的写者是否被新读者长期挤压，还取决于具体策略。[EV-S1](https://man7.org/linux/man-pages/man3/pthread_rwlock_rdlock.3p.html)


---

## LINUX-EVENT-002

**线程安全退出：请求停止、唤醒、join、清理**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-event-002)

**面试回答：**停止请求不等于线程已经退出。我会在锁内设置停止状态，禁止继续入队，唤醒各等待路径；消费者用 `while (count==0 && !stopping)` 等待，只有停止且队列为空才退出。控制线程解锁后 join 所有使用者，成功确认结束后才释放队列和同步对象。

**解析：**立即 free 会让仍在执行或即将恢复的线程访问失效对象。多个消费者都等待同一个条件变量时，停机通常使用 broadcast，让它们都检查状态；等待“队列不满”的生产者也不能遗漏。通知不会直接转交锁，线程醒来仍要重新竞争锁。只用 signal 可能留下未被唤醒的消费者，主线程随后一直等不到它退出。阻塞在设备读取上的线程还需单独的唤醒/取消等待设计。[EV-S2](https://man7.org/linux/man-pages/man3/pthread_join.3.html)[EV-S3](https://man7.org/linux/man-pages/man3/pthread_cond_wait.3p.html)


---

## LINUX-EVENT-003

**信号：终止请求不是资源计数**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-event-003)

**面试回答：**SIGTERM 允许程序实现有序终止，例如停止采集、处理已有任务、等待工作线程，再清理资源；SIGKILL 不允许应用接管终止过程。普通信号不按每次发送保留计数，同类信号在待处理期间重复产生可能合并，因此不适合用处理次数统计每帧事件。

**解析：**“可处理”不等于“自动清理”：没有实现处理策略时 SIGTERM 仍可能直接终止。内核回收进程资源和业务保存完成不是同一件事。标准信号适合表达“需要退出”这种幂等请求；逐帧工作应由队列或共享缓冲的生产消费协议记录。[EV-S4](https://man7.org/linux/man-pages/man7/signal.7.html)


---

## LINUX-EVENT-004

**异步信号处理与sigwait：收尾代码放在哪里**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-event-004)

**面试回答：**不能直接照搬，因为异步信号处理受到信号安全限制，pthread Mutex/条件变量调用不适合作为通用处理函数操作。可以在创建线程前屏蔽退出信号，由专门控制线程调用 sigwait 同步接收；等待返回以后是普通线程代码，再按正常同步规则执行停机。

**解析：**处理函数可能打断刚好持锁的线程，重拿锁会自锁。sigwait 的模型是线程主动等待后返回，不是异步插入的回调。要先设置屏蔽再创建线程，否则已有线程可能仍未屏蔽目标信号。这个协议并不免除普通线程中的锁顺序和资源生命周期责任。[EV-S5](https://man7.org/linux/man-pages/man7/signal-safety.7.html)[EV-S6](https://man7.org/linux/man-pages/man3/pthread_sigmask.3.html)，含sigwait线程示例)


---

## LINUX-EVENT-005

**消息队列与IPC选型：通知、字节流和完整消息**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-event-005)

**面试回答：**小型控制消息可用消息队列，保留边界并带有限排队能力；大图像可评估共享缓冲加同步，减少重复转交大块数据，但必须明确覆盖和回收规则。mq_receive 缓冲区太小会失败，而不是按字节流拆开同一消息。FIFO只是带名字的管道，不是持久存储。

**解析：**共享内存不替代通知和所有权管理；消息队列也不保证消费者再慢都不会满。POSIX接收缓冲区需满足配置的最大消息大小，不是仅比当前那条消息大就足够；不同消息优先级还影响接收顺序。题目不要求背全部IPC接口。[EV-S7](https://man7.org/linux/man-pages/man3/mq_receive.3.html)[EV-S8](https://man7.org/linux/man-pages/man3/mq_send.3.html)


---

## LINUX-EVENT-006

**I/O多路复用：先等谁就绪，再真正读写**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-event-006)

**面试回答：**线程被A的读取挡住，尚未执行到B。非阻塞避免停在某个read里，但无限重试会忙等。应通过I/O多路复用集中等待多个入口，就绪后再执行实际非阻塞读写，并保留每个连接的分帧和发送进度。

**解析：**等待函数返回的是“哪些入口值得处理”，不是收到的字节。其等待可以让线程睡眠，而非阻塞读写避免处理某个入口时再次卡住，两者并不矛盾。工作量很小或每线程只服务一个连接时，阻塞模型仍可以合理使用。[EV-S9](https://man7.org/linux/man-pages/man2/select.2.html)[EV-S10](https://man7.org/linux/man-pages/man2/poll.2.html)


---

## LINUX-EVENT-007

**select：关注名单返回后会变成结果名单**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-event-007)

**面试回答：**首参数是最大fd加1，即9。select把输入集合改成就绪结果集合，重复等待前必须恢复关注名单；Linux上超时结构也要重设。常见glibc集合要求描述符数值小于FD_SETSIZE，fd=1500即使连接很少也越界。

**解析：**maxfd+1规定检查的编号范围，不是对象数量。若一轮只有B就绪，直接复用结果集合会把A从下一轮关注名单中丢掉。使用FD_SET前应验证范围；较多或较大fd可以改用poll/epoll，而非越界写集合。[EV-S9](https://man7.org/linux/man-pages/man2/select.2.html)


---

## LINUX-EVENT-008

**poll：events写要求，revents读结果**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-event-008)

**面试回答：**返回1说明有一个数组元素包含结果事件；具体对象和类型要查看revents。events由程序指定，revents由内核返回。可读和挂断可能同时出现，应结合实际read结果处理剩余数据和EOF，而不是把两者当互斥事件。

**解析：**poll一次并不读入任何业务数据。循环应检查每个元素且使用位掩码，不以返回数当fd编号，也不把events误当当前状态。挂断表示方向关闭等状态变化，并不自动证明已缓冲的数据全部读取完。[EV-S10](https://man7.org/linux/man-pages/man2/poll.2.html)


---

## LINUX-EVENT-009

**epoll_create1与epoll_ctl：先建管理入口，再登记**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-event-009)

**面试回答：**epfd引用epoll管理实例，业务fd引用设备或Socket。应用在登记时设置关注事件和data附带信息，内核返回相应事件时带回。登记成功后，修改本地ev再登记B不改变A的记录。DEL只移除关注；对象是否关闭和其他引用是否存在是独立的资源管理问题。

**解析：**关注名单保存于内核，ev不是内核长期持有的那一个栈上结构体。若data里保存指针，内核保存的是指针值，不会替应用延长被指向对象的生命周期。此处不展开复杂多线程fd复用方案。[EV-S11](https://man7.org/linux/man-pages/man2/epoll_ctl.2.html)[EV-S13](https://man7.org/linux/man-pages/man7/epoll.7.html)


---

## LINUX-EVENT-010

**epoll_wait：返回的是事件结果，不是设备数组或图像**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-event-010)

**面试回答：**不会，结果数组大小限制的是单次取回数量，不是关注总量。结果按本次返回事件排列，应使用data识别来源，不能按登记次序推断下标。EPOLLIN是就绪通知，程序仍需执行读取、检查错误/EOF/EAGAIN，并根据协议组帧。

**解析：**只遍历n项有效结果，不能遍历全部8项并读取未填位置。当同一个fd有多种事件时按位检查；就绪不是给当前线程预留数据的锁，其他消费者可能改变可读状态。[EV-S12](https://man7.org/linux/man-pages/man2/epoll_wait.2.html)[EV-S13](https://man7.org/linux/man-pages/man7/epoll.7.html)


---

## LINUX-Q17

**LT与ET：同一批数据没读完，还会不会再次提醒**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-q17)

**面试回答：**ET处理一次事件后如果只读100字节就重新等待，可能错过继续处理缓冲区余下900字节的机会，表现为卡住。数据不一定丢了，只是程序在等不会因旧数据自动重报的事件。LT仍根据可读状态报告，所以未读余量通常会在后续等待中继续被报告。ET常用修复是非阻塞循环读取到EAGAIN，EOF和真正错误则分别处理。

**解析：**epoll_wait取走的是通知，read取走的才是数据。暂停发送但保持连接，可避免关闭事件或新数据让问题偶然“自行恢复”。自行验证时可以先部分读，再检查事件，最后不依赖新通知主动读出剩余900字节，以区分“没有新通知”和“没有数据”；原材料所述epoll实验程序未随包提供，本次未运行。不要把这个场景扩大成ET永远只通知一次。

**来源边界：**题干保留仓库原有编号和含义；LT对照及实验是本次教学补充，不是用户已答错记录。[EV-S13](https://man7.org/linux/man-pages/man7/epoll.7.html)


---

## LINUX-EVENT-011

**ET读循环：非阻塞读取到EAGAIN，再等下一次机会**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-event-011)

**面试回答：**只读一次可能留下数据却等不到新的ET提醒；循环读取可以耗尽当前可用输入。fd还必须非阻塞，否则耗尽后再读会把事件线程卡住。EAGAIN表示暂时无数据，保留连接和未完成帧状态、回去等待；流Socket读取0则表示接收方向结束，执行相应生命周期处理。

**解析：**不能以循环固定次数代替耗尽检测，也不能把短读误当完整包。EINTR是调用被中断，不应默认视作断线。若高流量要求限制预算，需自己保留待继续服务状态；不要在还有已知可处理工作时无期限睡回内核。[EV-S13](https://man7.org/linux/man-pages/man7/epoll.7.html)[EV-S14](https://man7.org/linux/man-pages/man2/read.2.html)


---

## LINUX-EVENT-012

**发送与EPOLLOUT：有待发数据才需要等可写**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-event-012)

**面试回答：**没有待发数据却关注常常就绪的可写事件，会造成无效唤醒或忙循环。400字节后EAGAIN应保留偏移400和剩余600字节，等待可写后从剩余位置继续。发送队列清空就应停止，不需要凑出一次EAGAIN；典型LT事件循环可以取消EPOLLOUT。

**解析：**可写是“有空间尝试提交”，不是“有新业务要发”。每次都从头write会重复前400字节。开启ET也不会替应用保存发送偏移、处理终止错误或实现可靠消息确认。[EV-S11](https://man7.org/linux/man-pages/man2/epoll_ctl.2.html)[EV-S15](https://man7.org/linux/man-pages/man2/write.2.html)


---

## LINUX-EVENT-013

**select、poll、epoll与LT/ET：性能不靠口号判断**

[返回例题](../嵌入式秋招八股文_Linux.md#q-linux-event-013)

**面试回答：**三个说法都过于绝对。epoll仍有登记、事件收集和逐项处理成本；ET和LT影响通知策略，吞吐取决于连接规模、批量读写、业务处理与调度方式；字节流消息边界由协议定义，不由事件划分。可以先使用易正确实现的LT，性能测量有依据后再决定ET及其配套状态管理。

**解析：**多个小I/O、长期持锁、同步日志和耗时解析可能才是真正瓶颈。ET降低某些重复通知的机会，但同样需要处理公平性、EAGAIN、关闭和部分消息。学会LT/ET行为不是已经完成服务器性能验证；不以文档阅读标记面试通过。[EV-S13](https://man7.org/linux/man-pages/man7/epoll.7.html)
