# FreeRTOS：知识点与例题

每个知识点独立展开，例题紧跟其后；点击“查看答案”直接到对应解析。代码段如未标“例题”，用于解释机制。整理不代表已经掌握；例题默认是学习题，原档面经出处另行注明。

## 目录

- [1. 任务状态](#1-任务状态)
- [2. 抢占、时间片与饥饿](#2-抢占时间片与饥饿)
- [3. 中断现场保护与任务上下文切换](#3-中断现场保护与任务上下文切换)
- [4. Cortex-M通用寄存器](#4-cortex-m通用寄存器)
- [5. MSP、PSP与EXC_RETURN](#5-msppsp与exc_return)
- [6. Cortex-M基本异常栈帧](#6-cortex-m基本异常栈帧)
- [7. 普通ISR与R4-R11保存责任](#7-普通isr与r4-r11保存责任)
- [8. Task Stack与TCB](#8-task-stack与tcb)
- [9. FPU上下文与lazy stacking](#9-fpu上下文与lazy-stacking)
- [10. PendSV的任务切换职责](#10-pendsv的任务切换职责)
- [11. PendSV为什么通常设为最低优先级](#11-pendsv为什么通常设为最低优先级)
- [12. 哪些事件会请求PendSV](#12-哪些事件会请求pendsv)
- [13. SysTick的职责](#13-systick的职责)
- [14. SysTick、PendSV与完整A→B切换](#14-systickpendsv与完整ab切换)
- [15. Queue的数据复制与阻塞语义](#15-queue的数据复制与阻塞语义)
- [16. Queue与“共享最新值+Mutex”](#16-queue与共享最新值mutex)
- [17. Binary Semaphore、Counting Semaphore与Mutex](#17-binary-semaphorecounting-semaphore与mutex)
- [18. 优先级反转与优先级继承](#18-优先级反转与优先级继承)
- [19. Task Notification](#19-task-notification)
- [20. Event Group](#20-event-group)
- [21. ISR与Task通信](#21-isr与task通信)
- [22. 中断优先级与任务优先级](#22-中断优先级与任务优先级)
- [23. Cortex-M优先级、BASEPRI与FromISR边界](#23-cortex-m优先级basepri与fromisr边界)
- [24. ISR不能阻塞](#24-isr不能阻塞)
- [25. Software Timer](#25-software-timer)
- [26. Critical Section](#26-critical-section)
- [27. Mutex、Queue、Critical Section与Atomic选型](#27-mutexqueuecritical-section与atomic选型)
- [28. 动态与静态任务创建](#28-动态与静态任务创建)
- [29. heap_1至heap_5](#29-heap_1至heap_5)
- [30. Task Stack Size单位](#30-task-stack-size单位)
- [31. Stack High Water Mark](#31-stack-high-water-mark)
- [32. Tick与`pdMS_TO_TICKS()`](#32-tick与pdms_to_ticks)
- [33. `vTaskDelay()`与`vTaskDelayUntil()`](#33-vtaskdelay与vtaskdelayuntil)
- [34. UART高吞吐的ISR→Buffer→Task架构](#34-uart高吞吐的isrbuffertask架构)
- [35. HardFault 定位](#35-hardfault-定位)
- [36. Watchdog + RTOS](#36-watchdog--rtos)
- [37. Priority Inversion / Deadlock](#37-priority-inversion--deadlock)
- [38. CPU 占用率 / 调度异常](#38-cpu-占用率--调度异常)
- [39. 系统“卡死”快速排查模板](#39-系统卡死快速排查模板)
- [40. 裸机迁移到 RTOS：重新划分执行与资源责任](#40-裸机迁移到-rtos重新划分执行与资源责任)

## 1. 任务状态

- **Running**：当前正在CPU上执行。
- **Ready**：具备运行条件，等待CPU。
- **Blocked**：等待时间或Queue、Semaphore、Notification等事件，不占CPU忙等。
- **Suspended**：被显式挂起，通常必须显式恢复。
- Blocked任务等到事件后先变为Ready，是否马上Running取决于调度结果。

## 2. 抢占、时间片与饥饿

- 抢占解决不同优先级任务的调度：更高优先级任务Ready后可抢占当前任务。
- 时间片解决同优先级Ready任务之间的轮转，受抢占与时间片配置影响。
- 高优先级任务若一直不阻塞，低优先级任务可能starvation；无工作时应等待事件，而非持续忙循环。

## 3. 中断现场保护与任务上下文切换

- 中断现场保护的目标是让被ISR打断的执行流能够恢复；普通ISR结束后通常继续原执行流。
- 任务上下文切换要冻结Task A的可恢复状态、选择并恢复Task B，使CPU真正换一个任务运行。
- 如果ISR唤醒更高优先级任务，异常退出路径可能先进入PendSV切换任务，不一定立即回到原任务。
- 不能只回答“一个保存得少、一个保存得多”，核心差异是是否改变当前运行任务。

### Q-RTOS-E01

**例题：中断现场保护 vs Task Context Switch**

解释两者的核心区别，并说明 Cortex-M 异常入口硬件自动保存哪些寄存器。

[查看答案](./答案/FreeRTOS答案.md#rtos-e01)

## 4. Cortex-M通用寄存器

- `R0-R3`常用于参数、返回值和临时计算，通常为caller-saved。
- `R4-R11`按AAPCS通常为callee-saved，被调用函数若修改应负责恢复。
- `R12`是临时寄存器；`R13=SP`指向栈顶；`R14=LR`保存函数或异常返回关系；`R15=PC`决定执行位置。
- `xPSR`保存条件标志、Thumb状态和异常状态等。HardFault分析中，堆栈中的PC/LR/xPSR是关键定位证据。

## 5. MSP、PSP与EXC_RETURN

- Cortex-M提供MSP和PSP。FreeRTOS典型端口中，Thread Mode的任务使用PSP，Handler Mode的ISR使用MSP。
- 异常入口的基本栈帧压到异常发生前所使用的栈；任务从PSP运行时，该帧通常落在任务栈中。
- 异常处理时LR可保存特殊`EXC_RETURN`编码，用于描述返回模式、返回后使用MSP/PSP以及是否涉及扩展浮点栈帧。

## 6. Cortex-M基本异常栈帧

异常入口时，硬件基本栈帧自动保存：

```text
R0-R3、R12、LR、PC、xPSR
```

`R4-R11`不属于基本硬件自动压栈集合；具体器件还可能受FPU、lazy stacking、对齐和异常模型影响。

**易错提醒**：曾把`R4-R11`也说成硬件异常入口自动保存。

### Q-FREERTOS-D-005

**例题：Cortex-M异常入口自动保存哪些寄存器？**

Cortex-M 基本异常帧是哪8个寄存器槽位？R4-R11是否在其中？区分栈帧内的LR与异常处理器当前LR中的 EXC_RETURN。

[查看答案](./答案/FreeRTOS答案.md#freertos-d-005)

## 7. 普通ISR与R4-R11保存责任

普通C ISR若使用`R4-R11`，编译器会按照ABI及实际寄存器使用生成必要的保存/恢复代码；硬件不会因为“ISR最后返回原任务”就自动扩展基本栈帧。编写裸汇编或特殊naked处理函数时，保存责任必须由程序员与端口代码明确承担。

**易错提醒**：曾认为普通ISR返回原任务，所以硬件会自动保存`R4-R11`。

### Q-FREERTOS-D-006

**例题：普通ISR与PendSV如何保护R4-R11？**

普通C ISR、naked汇编ISR、PendSV任务切换，分别由谁负责R4-R11的保存恢复？为什么普通ISR按实际使用保护，而切换必须保留任务所需全部状态？

[查看答案](./答案/FreeRTOS答案.md#freertos-d-006)

## 8. Task Stack与TCB

- 具体寄存器上下文主要保存在任务自己的Task Stack中。
- TCB保存`pxTopOfStack`、优先级、状态/调度链表节点、任务名等内核管理信息，并通过栈顶指针定位已保存上下文。
- 不能概括为“全部上下文直接保存在TCB”，也不能忽略端口可能在TCB中增加MPU、TLS等字段。

### Q-RTOS-E03

**例题：TCB vs Task Stack**

TCB 和 Task Stack 分别保存什么？为什么不能简单说“上下文保存在 TCB”？

[查看答案](./答案/FreeRTOS答案.md#rtos-e03)

## 9. FPU上下文与lazy stacking

- 带FPU的MCU中，使用浮点的任务可能还需保存浮点上下文；部分Cortex-M4F/M7F端口会处理`S16-S31`等寄存器。
- FPU上下文典型地仍保存在Task Stack，TCB主要记录管理信息和栈顶位置。
- 保存范围取决于CPU、FreeRTOS port、浮点ABI和lazy stacking；浮点调用也会增加任务栈最坏需求。

**易错提醒**：曾认为FPU上下文直接保存在TCB。

### Q-FREERTOS-D-007

**例题：FPU上下文保存在何处？**

在 Cortex-M4F 典型端口中，FPU 上下文在 TCB 还是任务栈？画出 TCB.pxTopOfStack 到任务栈的关系，并解释 lazy stacking 为什么不能用来忽略最坏栈消耗。

[查看答案](./答案/FreeRTOS答案.md#freertos-d-007)

## 10. PendSV的任务切换职责

PendSV统一执行任务上下文切换：保存当前任务尚未由硬件保存的上下文，将更新后的PSP写回当前TCB，调用调度器选择下一任务，再从下一TCB恢复其上下文并异常返回。

PendSV保存`R4-R11`不是因为PendSV代码本身一定使用它们，而是因为目标任务运行后会改变这些寄存器，必须完整冻结当前任务。

[FreeRTOS ARM_CM4F端口源码](https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/portable/GCC/ARM_CM4F/port.c)可核对PendSV、BASEPRI和浮点现场，其他架构须看自己的port。

### Q-FREERTOS-K-001

**例题：任务切换时保存和恢复了什么？**

以 Cortex-M4F 常见 FreeRTOS 端口为例，Task A 切到 B 时哪些信息由硬件入栈，哪些由端口保存，TCB 如何找到上下文？为什么不能只保存 PC？

原档出处：影石面经涉及 FreeRTOS 内核机制；该表述为同主题归纳，SRC-007；题干按学习需要明确化，本轮未重新核验面经，不作为逐字企业原题。

[查看答案](./答案/FreeRTOS答案.md#freertos-k-001)

## 11. PendSV为什么通常设为最低优先级

- 让高紧迫度外设ISR先完成，避免在嵌套中断中间切换Thread Mode任务。
- 将多个调度请求统一延后到异常链尾部处理，减少切换逻辑对中断响应的干扰。
- “低优先级”说的是NVIC异常优先级，不是FreeRTOS Task Priority。

## 12. 哪些事件会请求PendSV

- SysTick推进延时/超时或同优先级时间片后需要调度。
- 外设ISR唤醒更高优先级任务并请求切换。
- 当前任务调用`taskYIELD()`、`vTaskDelay()`或等待IPC进入Blocked。
- Queue、Semaphore、Notification等内核操作改变最高优先级Ready任务。

SysTick只是来源之一，不能把PendSV描述成“只由SysTick唤醒”。

**易错提醒**：曾把PendSV的请求来源缩减为SysTick。

### Q-FREERTOS-D-008

**例题：PendSV可能由哪些事件请求？**

分别判断以下事件是否可能请求PendSV：Tick使高优先级任务延时到期；同优先级时间片；ISR唤醒高优先级任务；taskYIELD；vTaskDelay；任务发队列唤醒更高优先级接收者。是否每次请求都一定选到另一任务？

[查看答案](./答案/FreeRTOS答案.md#freertos-d-008)

## 13. SysTick的职责

SysTick通常提供RTOS Tick时基：增加Tick计数、推进Delay/Timeout、将到期任务移到Ready集合、处理时间片需求，并在需要时请求PendSV。它主要回答“是否出现调度需求”，不负责完成全部任务寄存器切换。

## 14. SysTick、PendSV与完整A→B切换

典型流程：SysTick或外设事件使Task B变为Ready并请求PendSV；PendSV进入时硬件已把Task A的基本异常栈帧压栈，端口再保存`R4-R11`及必要FPU上下文，更新`A.TCB->pxTopOfStack`，选择B，恢复B的软件保存部分和PSP；异常返回时硬件恢复B的基本栈帧，B继续运行。

因此SysTick中的典型变化是Blocked→Ready并请求调度；Ready→Running由调度选择和PendSV切换实现。

**易错提醒**：曾把`Blocked→Ready→Running`全部理解成在SysTick ISR内部完成。

### Q-FREERTOS-D-009

**例题：SysTick与PendSV如何完成状态切换？**

单核抢占模式下 A 正在运行，优先级更高的 B 延时到期。按时间顺序说明 SysTick、Ready列表、PendSV、TCB栈顶指针、异常返回如何使 B Running，A 切走后是什么状态？

[查看答案](./答案/FreeRTOS答案.md#freertos-d-009)

### Q-RTOS-E02

**例题：SysTick vs PendSV**

解释 SysTick 和 PendSV 的职责区别，并完整说明高优先级延时 Task 到期后如何从 `Blocked` 变成 `Running`。

[查看答案](./答案/FreeRTOS答案.md#rtos-e02)

## 15. Queue的数据复制与阻塞语义

- Queue按创建时的item size复制内容，并通过满/空条件产生同步效果。
- 结构体元素复制结构体值；指针元素只复制地址值，所指对象生命周期和释放责任需另行设计。
- 队列可保存尚未消费的多条消息；队列深度不足或消费者过慢时，仍会阻塞、超时或丢数据。

## 16. Queue与“共享最新值+Mutex”

- 每条消息都要处理、需要保留历史顺序：Queue或Ring Buffer。
- 消费者只关心当前最新状态：共享状态加Mutex/Atomic，或长度1覆盖式快照队列。
- Mutex保护同一共享可变对象；Queue传递逐条消息。选择前先判断所有权、历史保留和整体一致性。

**易错提醒**：曾把“复杂共享配置→Queue、逐条完整消息→Mutex”回答反了。

### Q-FREERTOS-D-010

**例题：Mutex与Queue如何选型？**

四种需求分别怎么设计：多个任务同时修改复杂配置且字段需一致；逐帧处理不可漏的 SensorData；显示仅需最新温度；高速UART连续字节流。解释选择的消息历史、所有权及容量边界。

[查看答案](./答案/FreeRTOS答案.md#freertos-d-010)

## 17. Binary Semaphore、Counting Semaphore与Mutex

- Binary Semaphore偏事件通知，容量1时重复Give可能合并事件。
- Counting Semaphore偏资源数量或允许累计的同类事件，需评估计数上限。
- Mutex偏共享资源互斥，强调获取者释放并提供优先级继承；不能在ISR中获取Mutex。

### Q-RTOS-E04

**例题：Binary Semaphore vs Mutex**

两者有什么区别？为什么 Mutex 可以实现 Priority Inheritance？

[查看答案](./答案/FreeRTOS答案.md#rtos-e04)

## 18. 优先级反转与优先级继承

L持有Mutex，H请求后Blocked，M不断抢占L，导致H间接等待M，这叫Priority Inversion。Mutex的Priority Inheritance让L临时继承H优先级，尽快释放Mutex后恢复原优先级。继承不能解决死锁或无限长临界区。

**易错提醒**：曾把“Mutex支持优先级继承”说成“Mutex支持优先级反转”。

### Q-FREERTOS-D-001

**例题：为什么这个任务会发生优先级反转？**

H/M/L 任务优先级依次为3/2/1。L持锁后H请求该锁阻塞，此时M一直Ready。无继承和启用Mutex继承时分别谁运行？若A持M1等M2、B持M2等M1，继承能解除吗？

[查看答案](./答案/FreeRTOS答案.md#freertos-d-001)

### Q-RTOS-E05

**例题：Priority Inversion**

解释优先级反转，并用 H/M/L 三个任务描述 Priority Inheritance 的处理过程。

[查看答案](./答案/FreeRTOS答案.md#rtos-e05)

### Q-RTOS-D05

**例题：Priority Inversion**

给 H/M/L 三个 Task 及一个 Mutex，画出无继承和有继承情况下的调度时序。

[查看答案](./答案/FreeRTOS答案.md#rtos-d05)

## 19. Task Notification

Task Notification面向固定任务，可表达通知值、置位或计数，通常比另建Queue/Semaphore轻量，适合ISR→Task。它不适合多个接收者共享同一消息，也不天然携带任意长度数据；API动作的覆盖、增量或置位语义必须匹配丢事件要求。

## 20. Event Group

Event Group用多个bit表达多个布尔条件，可等待任一位或全部指定值，并选择退出时是否清位。它适合READY/ALARM组合，不适合保存连续温度值或每次事件快照；部分高位由内核保留，可用bit数需结合版本和Tick配置。

## 21. ISR与Task通信

- ISR使用非阻塞`...FromISR()` API，执行最小采集、清标志和通知，把协议解析、CRC、日志和复杂循环延后到任务。
- `xHigherPriorityTaskWoken`是由FromISR API更新的`BaseType_t`输出状态；为`pdTRUE`时在ISR退出路径请求切换。
- ISR执行过长会增加中断延迟、jitter、栈压力和FIFO溢出风险，并延迟Ready任务获得CPU。

### 后果
- 同级/低级 IRQ pending
- 高优先级 Ready Task 延迟运行
- ISR latency / jitter 增大
- FIFO/UART/DMA 数据丢失
- 嵌套中断增加栈压力

### 推荐结构
```text
ISR：
读状态 / 清标志 / 少量搬运
→ Buffer / DMA 状态
→ Notification / Queue
→ 退出

Task：
协议解析
CRC
状态机
日志
业务处理
```

### Q-RTOS-E08

**例题：ISR 唤醒高优先级任务**

```text
Task A priority = 2，当前 Running
Task B priority = 4，当前 Blocked
```
UART ISR 中执行 `xQueueSendFromISR()` 后使 B 变成 Ready。

完整描述 `UART IRQ 进入 → B 最终 Running`，说明 A/B 状态变化、`xHigherPriorityTaskWoken`、PendSV 和上下文切换时机。

[查看答案](./答案/FreeRTOS答案.md#rtos-e08)

### Q-RTOS-D04

**例题：ISR**

UART ISR 中做 CRC、printf、协议解析，偶发 ADC 采样 jitter。如何证明根因并重构？

[查看答案](./答案/FreeRTOS答案.md#rtos-d04)

## 22. 中断优先级与任务优先级

二者属于两套体系：NVIC Priority决定ISR之间的抢占；Task Priority决定Thread Mode中Ready任务的选择。只要硬件中断被允许响应，ISR可打断任务；任务不能靠更高Task Priority抢占正在执行的ISR。

## 23. Cortex-M优先级、BASEPRI与FromISR边界

- Cortex-M优先级数值通常越小，硬件紧迫度越高。
- FreeRTOS端口常用BASEPRI屏蔽达到阈值及更低紧迫度的中断，同时允许更高紧迫度中断响应。
- `configMAX_SYSCALL_INTERRUPT_PRIORITY`表示可调用内核ISR API的最高逻辑紧迫度边界：高于边界的高紧迫度ISR不受内核临界区屏蔽，因此不能调用FreeRTOS API。
- 判断时还需考虑NVIC实现位数、优先级分组和库宏是否已经完成位移编码。

本文BASEPRI示例限定具有该寄存器的常见Cortex-M3/M4/M7端口；M0/M0+等不能直接照搬。优先级分组必须满足所用port约束。

### Q-RTOS-E07

**例题：NVIC / BASEPRI / FromISR**

Cortex-M 有 4 个有效中断优先级位：

```text
0~15，数字越小逻辑优先级越高
configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY = 5
```

现有：

```text
UART IRQ = 2
DMA  IRQ = 5
ADC  IRQ = 9
```

回答：
1. 谁硬件中断优先级最高？
2. 哪些 ISR 可以调用 `xQueueSendFromISR()`？
3. FreeRTOS 临界区内哪些 ISR 仍可抢占？
4. 为什么这些仍可抢占的高优先级 ISR 反而不能调用 FreeRTOS 内核 API？

[查看答案](./答案/FreeRTOS答案.md#rtos-e07)

## 24. ISR不能阻塞

`vTaskDelay()`要把“当前任务”放入延时列表并切换调度，但ISR不是Task，没有任务级Blocked状态，也没有可供其延时恢复的TCB语义。因此ISR不能调用会阻塞的任务API，只能使用专门的非阻塞FromISR接口并尽快异常返回。

### Q-RTOS-E06

**例题：ISR 阻塞问题**

为什么 ISR 不能调用 `vTaskDelay()`，也不能 Take Mutex 后阻塞等待？

[查看答案](./答案/FreeRTOS答案.md#rtos-e06)

## 25. Software Timer

One-shot到期一次，Auto-reload周期重装。Timer Callback运行在Timer Service/Daemon Task而非硬件ISR，多个回调串行共享该任务；回调应短、快、避免阻塞，复杂工作通过Queue或Notification交给工作任务。Daemon优先级和Timer命令队列也会影响延迟。

## 26. Critical Section

- 适合极短、必须原子完成的指针或状态更新。
- 具体屏蔽机制由port决定；Cortex-M常通过优先级门槛屏蔽相关中断，并非一概关闭所有中断。
- 20 ms SPI/I2C事务、`printf`和复杂算法不应整体放入临界区；任务间共享外设通常使用Mutex。

**易错提醒**：曾把`taskENTER_CRITICAL()`绝对化为“关闭所有中断”。

### Q-FREERTOS-D-002

**例题：Critical Section到底屏蔽什么？**

在使用 BASEPRI 的 Cortex-M4 端口中，临界区是否屏蔽全部 IRQ？比较“保护与ISR共享的三个链表指针更新”和“两个任务共享20ms SPI事务”的同步方式；高于系统调用阈值的ISR访问同一链表又怎么办？

[查看答案](./答案/FreeRTOS答案.md#freertos-d-002)

## 27. Mutex、Queue、Critical Section与Atomic选型

- 多个任务直接访问同一复杂对象并要求字段整体一致：Mutex。
- 生产者产生完整消息、消费者逐条处理：Queue。
- 极短且不能被相关中断/调度打断的更新：Critical Section。
- 单个独立标志或计数的原子读改写：架构/工具链支持下使用Atomic；多个字段的一致性不能由若干独立Atomic自动保证。

### Q-RTOS-E10

**例题：机制选型**

为以下场景选择最合理机制并解释原因：
- A. 两个 Task 共享 I2C
- B. ISR 只需通知固定 Task“数据已到”
- C. 5 个 DMA Buffer 组成资源池
- D. 一帧帧 SensorData，每帧不能漏
- E. DisplayTask 只读取最新温度
- F. 极短链表指针修改

可选：Mutex / Binary Semaphore / Counting Semaphore / Queue / Task Notification / Critical Section / Atomic / 共享值+Mutex

[查看答案](./答案/FreeRTOS答案.md#rtos-e10)

## 28. 动态与静态任务创建

`xTaskCreate()`通常从FreeRTOS heap动态分配TCB与Task Stack，必须检查失败；`xTaskCreateStatic()`由用户提供`StaticTask_t`和`StackType_t`数组，便于确定内存大小和位置，但缓冲区生命周期必须覆盖任务。

## 29. heap_1至heap_5

- `heap_1`：只分配、不释放，最简单且确定性较好。
- `heap_2`：支持释放，但不合并相邻空闲块，容易碎片化，通常不作为新设计首选。
- `heap_3`：封装标准`malloc/free`，线程安全和行为依赖库与端口配置。
- `heap_4`：支持释放并合并相邻空闲块，适合单一连续heap区。
- `heap_5`：类似heap_4，但支持多个不连续RAM区域，需先定义内存区域。
- `xPortGetFreeHeapSize()`与历史最小heap只能说明容量指标，不能替代碎片和申请失败分析。

## 30. Task Stack Size单位

`xTaskCreate()`的栈深度通常是`StackType_t`元素个数，不是byte。若`sizeof(StackType_t)=4`且深度为512，名义栈容量约2048 byte；仍应以实际端口类型、对齐和API文档为准。

## 31. Stack High Water Mark

HWM是任务历史运行中最少剩余的未使用栈，通常也以`StackType_t`元素计；越小越接近溢出。测量应覆盖最深调用、局部大数组、格式化输出、FPU和异常路径，并保留余量；不能把HWM与未分配heap相加。

**易错提醒**：曾把HWM模糊表述为“剩余使用栈空间”。

### 高频现象
- 随机 HardFault
- 任务莫名卡死
- 返回地址被破坏
- 某些功能一开启才崩
- `printf/sprintf`、大数组、深调用后更容易出现

### 高频原因
- Task Stack 配置过小
- 大局部数组
- 深层函数调用 / 递归
- 格式化输出
- FPU Context
- 第三方库栈消耗大

### 排查手段
1. `uxTaskGetStackHighWaterMark()`
2. `configCHECK_FOR_STACK_OVERFLOW`
3. `vApplicationStackOverflowHook()`
4. 栈填充值 / Canary
5. 覆盖最坏业务路径长期运行
6. 检查调用链与局部变量

### 必须会解释
- High Water Mark = 历史最小剩余未使用栈
- Heap Free ≠ Task Stack Remaining
- Heap 还有很多，不代表已创建 Task 的 Stack 不会溢出
- 大数组改 `static` 能减栈，但会引入共享、不可重入和常驻 RAM 风险

填充扫描得到的HWM可能受未写入局部数组、优化和采样覆盖影响，不是数学上的最坏栈界。栈Hook也不能覆盖所有瞬时溢出，必须结合静态分析和故障现场。

### Q-FREERTOS-D-003

**例题：如何解释和使用Stack High Water Mark？**

假设 StackType_t=4字节，四任务的栈深度/HWM分别为 A:1024/600、B:512/12、C:256/80、D:512/300；A/D只测过空载，B有1200字节局部数组和sprintf，C跑过主要分支。哪项最危险？能否据此直接缩小A/D栈？heap还有25KB是否有帮助？

[查看答案](./答案/FreeRTOS答案.md#freertos-d-003)

### Q-RTOS-E12

**例题：Task Stack 风险**

```text
Task A:
StackDepth = 1024 words
HighWaterMark = 600

Task B:
StackDepth = 512 words
HighWaterMark = 12
内部有 uint8_t buf[1200]
并频繁 sprintf()

FreeRTOS Heap Free = 25 KB
```
回答：
1. 哪个 Task 更危险？
2. Heap 还有 25 KB 能不能证明 B 安全？
3. 如何进一步排查？
4. 把 `buf` 改成 `static` 有什么收益和风险？

[查看答案](./答案/FreeRTOS答案.md#rtos-e12)

### Q-RTOS-D01

**例题：栈**

Task B HighWaterMark=8，Heap Free=30KB。能否说明 Task B 安全？如何进一步验证？

[查看答案](./答案/FreeRTOS答案.md#rtos-d01)

## 32. Tick与`pdMS_TO_TICKS()`

`pdMS_TO_TICKS(ms)`把毫秒转换为Tick。1 Tick约为`1000/configTICK_RATE_HZ`毫秒，实际还受整数取整、Tick宽度和宏实现影响；小于一个Tick的非零时间可能转成0，Tick延时不能替代微秒硬件定时。

**易错提醒**：曾把该宏说成“把Tick转成时间”。

### Q-FREERTOS-D-004

**例题：pdMS_TO_TICKS与Tick换算**

采用整数向下取整的 pdMS_TO_TICKS，100Hz和1000Hz下分别换算1ms、10ms、25ms。为什么相对延时任务会漂移？周期到期是否等于立刻执行？

[查看答案](./答案/FreeRTOS答案.md#freertos-d-004)

## 33. `vTaskDelay()`与`vTaskDelayUntil()`

`vTaskDelay()`从调用点进行相对延时，任务执行时间会叠加到周期；`vTaskDelayUntil()`按计划唤醒基准推进，可减少周期累计漂移。到期只保证任务Ready，不保证立即Running；若执行已超期，应分析WCET、抖动和连续补跑，而非用延时API掩盖问题。

### Q-RTOS-E09

**例题：1ms 周期任务**

```c
while (1)
{
    Control();
    vTaskDelay(pdMS_TO_TICKS(1));
}
```
`Control()` 平均执行 0.3 ms。说明问题，并给出更合理设计。

[查看答案](./答案/FreeRTOS答案.md#rtos-e09)

## 34. UART高吞吐的ISR→Buffer→Task架构

逐字节`xQueueSendFromISR()`会产生频繁内核调用、数据复制、队列管理和任务唤醒。高吞吐场景更常使用UART DMA或硬件FIFO搬运到Ring/Stream Buffer，ISR只清标志、记录生产位置并发Task Notification，Parser Task批量处理。仍需设计环形缓冲区并发边界、DMA写指针快照、缓存一致性、溢出计数和背压策略。

### 典型低效结构
```text
每 Byte IRQ
→ xQueueSendFromISR()
→ 每 Byte 唤醒/调度
```

### 高吞吐结构
```text
UART + DMA
↓
Circular / Ping-Pong Buffer
↓
ISR 只更新位置 + Notification
↓
Parser Task 批量处理
```

### 面试追问
- DMA Buffer 覆盖怎么办？
- Producer > Consumer 怎么办？
- Ring Buffer 如何判断满/空？
- 丢帧策略是什么？
- 是否需要 backpressure？

### Q-RTOS-E11

**例题：UART 高吞吐重构**

UART 2 Mbps，当前每收到一个 Byte 都执行：
```c
xQueueSendFromISR(queue, &ch, ...);
```
CPU 占用明显过高。设计 `UART / DMA / Ring Buffer / Task` 架构，并说明 ISR 做什么、Task 做什么、为什么性能提高、Ring Buffer 满时如何处理。

[查看答案](./答案/FreeRTOS答案.md#rtos-e11)

### Q-RTOS-D06

**例题：Ring Buffer**

UART DMA 写 Ring Buffer，ParserTask 偶发处理不过来。如何判断是 Buffer 太小还是消费性能不足？满时策略有哪些？

[查看答案](./答案/FreeRTOS答案.md#rtos-d06)

## 35. HardFault 定位

### 第一优先级现场
- stacked `PC`
- stacked `LR`
- `SP`
- `xPSR`

### Fault 状态
- `CFSR`
- `HFSR`
- `BFAR`
- `MMFAR`

### 典型流程
```text
获取 PC/LR/SP
↓
读取 CFSR/HFSR
↓
若地址寄存器有效，看 BFAR/MMFAR
↓
PC 映射到 ELF/map/反汇编
↓
确认当前 Task
↓
检查 Task Stack
↓
检查指针、数组越界、函数指针、非法地址
```

### 常见根因
- NULL / 野指针
- 数组越界
- Stack Overflow
- 错误函数指针
- 非法 PC/LR
- 访问非法外设/内存地址
- 错误异常返回
- 未对齐访问（与配置/指令相关）

### Q-RTOS-E13

**例题：HardFault**

系统偶发 HardFault，可以获取：
`PC / LR / SP / xPSR / CFSR / HFSR / BFAR / MMFAR`

给出定位顺序，并说明这些寄存器分别能提供什么关键信息。

[查看答案](./答案/FreeRTOS答案.md#rtos-e13)

### Q-RTOS-D02

**例题：HardFault**

stacked PC 指向 `memcpy()` 内部。如何判断是 `memcpy()` 自己的问题，还是传入指针/长度已经错误？

[查看答案](./答案/FreeRTOS答案.md#rtos-d02)

## 36. Watchdog + RTOS

### 错误设计
```text
WatchdogTask 独立周期喂狗
```
只要 WatchdogTask 还活着，即使其他关键 Task 已死锁，系统仍不会复位。

### 推荐设计
```text
SensorTask ─┐
CommTask   ─┼→ 更新心跳
ControlTask ─┘

MonitorTask
↓
检查所有关键 Task 是否在时间窗口内健康
↓
全部健康 → Feed HW WDG
异常 → 不 Feed / 保存现场
```

### 卡死但没复位的排查
- Watchdog 是否真正启动
- Debug 模式是否冻结 WDG
- 是否有多个喂狗点
- ISR/错误 Task 是否仍在喂狗
- Timeout 是否配置错误
- Reset Cause 是否正确读取
- 调度 Tick 是否仍增长
- 各 Task 心跳是否更新

健康应指实际工作推进，不是任务还在空循环；各任务采用符合业务的独立时间窗口。只保留一个受健康汇总控制的喂狗入口，避免其他路径掩盖故障。

### Q-RTOS-E14

**例题：Watchdog 失效**

CommTask 已死锁 30 秒，但硬件 Watchdog 一直没有 Reset。列出至少 4 个可能原因，并给出更合理的 RTOS Watchdog 设计。

[查看答案](./答案/FreeRTOS答案.md#rtos-e14)

### Q-RTOS-D03

**例题：Watchdog**

ControlTask 死锁，但系统持续喂狗。设计一个能够检测“单任务失活”的 Watchdog 架构。

[查看答案](./答案/FreeRTOS答案.md#rtos-d03)

## 37. Priority Inversion / Deadlock

### Priority Inversion
H 等待 L 持有的 Mutex，而 M 抢占 L。

### Priority Inheritance
临时提升 L 优先级，使其尽快释放 Mutex。

### Deadlock
Priority Inheritance 不能解决：
```text
A holds M1, waits M2
B holds M2, waits M1
```

### Debug 思路
- 找谁持锁
- 找谁在等锁
- 看锁获取顺序
- 看持锁时间
- 看任务优先级
- Trace 调度和 Mutex wait

## 38. CPU 占用率 / 调度异常

### 方法
- Idle Task 比例
- FreeRTOS Run-Time Stats
- SystemView / Tracealyzer / IDE RTOS Viewer

### 不能只看 CPU%
需要结合：
- Deadline
- ISR latency
- Task response time
- Idle margin
- Context switch frequency

运行统计计时源须有足够分辨率；ISR时间可能被计入被打断任务，不能把工具百分比直接当纯任务计算时间。平均CPU低也可能因一次长ISR而错过截止期。

## 39. 系统“卡死”快速排查模板

```text
1. CPU 还在跑吗？
   → PC 是否变化 / Tick 是否增长

2. 卡在 Thread Mode 还是 Handler Mode？
   → 是否长 ISR

3. 当前 Task 是谁？
   → Task list / trace

4. 各 Task 状态？
   → Ready / Blocked / Suspended

5. 是否死锁？
   → Mutex owner / waiter

6. 是否栈溢出？
   → High Water Mark / Hook

7. 是否内存失败？
   → Heap free / malloc failed hook

8. 是否 Fault？
   → PC/LR/CFSR/HFSR/BFAR/MMFAR

9. Watchdog 为什么没处理？
   → Feed 设计 / reset cause
```

## 40. 裸机迁移到 RTOS：重新划分执行与资源责任

迁移不只是把 while(1) 放进任务。先画事件来源、处理链、数据所有者和最大时限，再决定任务划分。

- 按阻塞需求和响应期限分任务；任务数量不是功能函数数量，优先级不按代码大小设置。
- 把忙等和 delay_ms 改成带超时的事件等待/周期释放；驱动是否异步及可重入必须重新审查。
- 全局对象明确单一所有者，任务间用队列交接数据或互斥保护一致性；ISR 只用合法优先级的 FromISR 接口。
- 重新预算每个任务栈、ISR 的 MSP、堆和 DMA 缓冲区；检查创建失败及清理责任。
- 启动调度前准备驱动/IPC，避免中断先访问未创建的队列；设计背压、死锁排查与逐任务健康监控。

### Q-RTOS-E15

**例题：裸机 → RTOS**

将 `while(1)轮询 + UART ISR + delay_ms() + 大量全局变量` 的裸机工程迁移到 FreeRTOS，列出至少 5 个必须重新设计或审查的地方。

[查看答案](./答案/FreeRTOS答案.md#rtos-e15)
