# Lab6 实验报告：进程调度

## 实验概述

本实验在 Lab5 的基础上，实现了完整的进程调度框架和 Round Robin（RR）调度算法。通过本次实验，深入理解了操作系统的调度机制，包括调度类的抽象设计、运行队列的管理以及时间片轮转调度算法的实现。

---

## 练习0：填写已有实验

已成功将 Lab2/3/4/5 的代码填入 Lab6 相应位置，并进行了以下关键更新以支持调度功能：

### 1. 进程控制块初始化更新

在 `kern/process/proc.c` 的 `alloc_proc()` 函数中，添加了 Lab6 调度相关字段的初始化：

```c
// LAB6:YOUR CODE (update LAB5 steps)
proc->rq = NULL;                                    // 运行队列指针初始化为NULL
list_init(&(proc->run_link));                       // 初始化运行队列链表节点
proc->time_slice = 0;                               // 时间片初始化为0
proc->lab6_run_pool.left = proc->lab6_run_pool.right = proc->lab6_run_pool.parent = NULL;
proc->lab6_stride = 0;                              // stride值初始化为0
proc->lab6_priority = 0;                            // 优先级初始化为0
```

**修改原因**：Lab6 在进程控制块中新增了调度相关字段（运行队列指针、时间片、stride 调度所需的堆节点和优先级等），必须在进程创建时正确初始化这些字段，否则会导致调度器访问未初始化的内存，造成系统崩溃。

### 2. 时钟中断处理更新

在 `kern/trap/trap.c` 中的时钟中断处理函数中，添加了调度器的时钟处理：

```c
// 设置下一次时钟中断
clock_set_next_event();

// 时钟中断计数器加一
ticks++;

// 每100次时钟中断处理一次
if (ticks % TICK_NUM == 0) {
    print_ticks();
}

// LAB6: 调用调度器的时钟中断处理函数
sched_class_proc_tick(current);
```

**修改原因**：Lab6 的调度算法需要在每次时钟中断时更新当前进程的时间片，并在时间片耗尽时触发进程切换。如果不调用 `sched_class_proc_tick()`，进程将永远占用 CPU，无法实现时间片轮转。

### 3. 移除 DEBUG_GRADE 的早期 panic

修改了 `print_ticks()` 函数，移除了 DEBUG_GRADE 模式下的早期 panic：

```c
static void print_ticks()
{
    cprintf("%d ticks\n", TICK_NUM);
    // 移除了 DEBUG_GRADE 的 panic，允许测试程序完整运行
}
```

**修改原因**：原来的实现会在 100 ticks 后立即 panic 退出，但 priority 测试程序需要 2000ms 才能完成，导致测试无法通过。

---

## 练习1：理解调度器框架的实现

### 1.1 调度类结构体 sched_class 的分析

`sched_class` 结构体定义在 `kern/schedule/sched.h` 中，是调度算法的抽象接口：

```c
struct sched_class {
    const char *name;                                        // 调度器名称
    void (*init)(struct run_queue *rq);                     // 初始化运行队列
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);  // 进程入队
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);  // 进程出队
    struct proc_struct *(*pick_next)(struct run_queue *rq); // 选择下一个进程
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc); // 时钟中断处理
};
```

#### 各函数指针的作用和调用时机

| 函数指针 | 作用 | 调用时机 |
|---------|------|---------|
| `init` | 初始化运行队列，设置队列的初始状态 | 系统启动时，`sched_init()` 中调用 |
| `enqueue` | 将进程加入运行队列，按照调度算法的规则插入 | 进程变为 RUNNABLE 状态时，在 `wakeup_proc()` 中调用 |
| `dequeue` | 将进程从运行队列移除 | 进程被选中运行时，在 `schedule()` 中调用 |
| `pick_next` | 根据调度算法选择下一个要运行的进程 | 需要进行进程切换时，在 `schedule()` 中调用 |
| `proc_tick` | 处理时钟中断，更新时间片等调度信息 | 每次时钟中断时，在 `interrupt_handler()` 中调用 |

#### 为什么使用函数指针

使用函数指针而非直接实现函数的原因：

1. **实现多态性**：允许在运行时动态选择不同的调度算法（RR、Stride、FIFO 等），而无需修改调度框架的核心代码。

2. **解耦设计**：调度框架（`sched.c`）只依赖于抽象接口（`sched_class`），不依赖于具体的调度算法实现。这符合面向对象设计中的"依赖倒置原则"。

3. **易于扩展**：添加新的调度算法只需：
   - 实现 `sched_class` 接口的所有函数
   - 在 `sched_init()` 中切换 `sched_class` 指针
   - 无需修改调度框架的任何代码

4. **代码复用**：`sched.c` 中的 `wakeup_proc()`、`schedule()` 等函数可以被所有调度算法共用。

### 1.2 运行队列结构体 run_queue 的分析

Lab6 的 `run_queue` 结构体：

```c
struct run_queue {
    list_entry_t run_list;           // 链表形式的运行队列
    unsigned int proc_num;            // 队列中进程数量
    int max_time_slice;               // 最大时间片
    skew_heap_entry_t *lab6_run_pool; // 斜堆形式的运行队列（Lab6 新增）
};
```

#### Lab5 vs Lab6 的差异

| 特性 | Lab5 | Lab6 |
|-----|------|------|
| 数据结构 | 只有链表（`run_list`） | 链表 + 斜堆（`lab6_run_pool`） |
| 调度算法 | 简单的 FIFO | 支持 RR 和 Stride |
| 复杂度 | 入队/出队 O(1) | RR: O(1), Stride: O(log n) |

#### 为什么需要两种数据结构

1. **链表（list）适用于 RR 调度**：
   - RR 算法按 FIFO 顺序调度进程
   - 链表支持 O(1) 时间复杂度的队尾插入和队头删除
   - 实现简单，内存开销小

2. **斜堆（skew_heap）适用于 Stride 调度**：
   - Stride 算法需要按 stride 值（优先级）选择进程
   - 斜堆是一种自适应的优先级队列，支持 O(log n) 的插入和最小值查找
   - 相比普通堆，斜堆不需要维护完全二叉树的平衡性，实现更简单

3. **灵活性考虑**：
   - 不同调度算法可以选择最适合的数据结构
   - 提高了调度器的效率和可扩展性

### 1.3 调度器框架函数分析

#### sched_init() 函数

```c
void sched_init(void)
{
    list_init(&timer_list);
    
    sched_class = &default_sched_class;  // 设置调度类
    
    rq = &__rq;
    rq->max_time_slice = MAX_TIME_SLICE;
    sched_class->init(rq);               // 初始化运行队列
    
    cprintf("sched class: %s\n", sched_class->name);
}
```

**作用**：初始化调度器，设置全局调度类和运行队列。

**与具体算法的解耦**：通过 `sched_class` 指针间接调用具体算法的 `init()` 函数，框架代码不需要知道具体是哪种调度算法。

#### wakeup_proc() 函数

```c
void wakeup_proc(struct proc_struct *proc)
{
    assert(proc->state != PROC_ZOMBIE);
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        if (proc->state != PROC_RUNNABLE)
        {
            proc->state = PROC_RUNNABLE;
            proc->wait_state = 0;
            if (proc != current)
            {
                sched_class_enqueue(proc);  // 通过函数指针入队
            }
        }
    }
    local_intr_restore(intr_flag);
}
```

**作用**：唤醒进程，将其状态设置为 RUNNABLE 并加入运行队列。

**与具体算法的解耦**：通过 `sched_class_enqueue()` 间接调用，不同的调度算法有不同的入队策略（RR 加到队尾，Stride 按优先级插入）。

#### schedule() 函数

```c
void schedule(void)
{
    bool intr_flag;
    struct proc_struct *next;
    local_intr_save(intr_flag);
    {
        current->need_resched = 0;
        if (current->state == PROC_RUNNABLE)
        {
            sched_class_enqueue(current);     // 当前进程重新入队
        }
        if ((next = sched_class_pick_next()) != NULL)  // 选择下一个进程
        {
            sched_class_dequeue(next);         // 将选中的进程出队
        }
        if (next == NULL)
        {
            next = idleproc;
        }
        next->runs++;
        if (next != current)
        {
            proc_run(next);                    // 切换进程
        }
    }
    local_intr_restore(intr_flag);
}
```

**作用**：执行进程调度，选择下一个进程并切换上下文。

**与具体算法的解耦**：通过 `pick_next()` 函数指针，由具体的调度算法决定选择哪个进程运行。

### 1.4 调度类的初始化流程

完整的初始化流程：

```
kern_init()                     // 内核入口 (kern/init/init.c)
    └─> sched_init()            // 初始化调度器 (kern/schedule/sched.c)
        ├─> list_init(&timer_list)
        ├─> sched_class = &default_sched_class  // 设置为 RR_scheduler
        ├─> rq = &__rq
        ├─> rq->max_time_slice = 5
        └─> sched_class->init(rq)  // 调用 RR_init()
            └─> RR_init(rq)         // (kern/schedule/default_sched.c)
                ├─> list_init(&(rq->run_list))
                └─> rq->proc_num = 0
```

**关联机制**：
- `default_sched_class` 是一个全局的 `sched_class` 结构体实例
- 在 `default_sched.c` 中定义并初始化了所有函数指针
- `sched_init()` 将全局指针 `sched_class` 指向 `default_sched_class`

### 1.5 进程调度流程

#### 流程图

```
时钟中断触发
    ↓
trap() [kern/trap/trap.c]
    ↓
trap_dispatch()
    ↓
interrupt_handler()
    ↓
IRQ_S_TIMER 分支
    ↓
clock_set_next_event()  ← 设置下一次时钟中断
    ↓
ticks++
    ↓
sched_class_proc_tick(current)  ← 调用调度类的时钟处理函数
    ↓
RR_proc_tick(rq, proc)  [kern/schedule/default_sched.c]
    ├─> proc->time_slice--
    └─> if (time_slice == 0)
            proc->need_resched = 1  ← 设置需要调度标志
    ↓
trap 返回前检查 need_resched
    ↓
if (current->need_resched)
    ↓
schedule()  [kern/schedule/sched.c]
    ├─> current->need_resched = 0
    ├─> if (current->state == PROC_RUNNABLE)
    │       sched_class_enqueue(current)  → RR_enqueue()
    ├─> next = sched_class_pick_next()    → RR_pick_next()
    ├─> sched_class_dequeue(next)         → RR_dequeue()
    └─> proc_run(next)
        └─> switch_to() ← 切换上下文
```

#### need_resched 标志的作用

1. **触发调度的信号**：
   - 时间片耗尽时设置为 1
   - 进程主动放弃 CPU（`do_yield()`）时设置为 1
   - 高优先级进程唤醒时设置为 1

2. **延迟调度机制**：
   - 不在中断处理函数中直接调度（避免嵌套中断的复杂性）
   - 在中断返回前检查标志，安全地进行进程切换

3. **防止重复调度**：
   - `schedule()` 函数开始时清除标志
   - 避免同一个调度请求被处理多次

### 1.6 调度算法的切换机制

#### 添加新调度算法（如 Stride）的步骤

1. **实现 sched_class 接口**（`kern/schedule/stride_sched.c`）：
```c
struct sched_class stride_sched_class = {
    .name = "stride_scheduler",
    .init = stride_init,
    .enqueue = stride_enqueue,
    .dequeue = stride_dequeue,
    .pick_next = stride_pick_next,
    .proc_tick = stride_proc_tick,
};
```

2. **修改 sched_init() 切换调度类**（`kern/schedule/sched.c`）：
```c
void sched_init(void)
{
    // 只需修改这一行
    sched_class = &stride_sched_class;  // 从 default_sched_class 改为 stride_sched_class
    
    // 其他代码不变
    rq = &__rq;
    rq->max_time_slice = MAX_TIME_SLICE;
    sched_class->init(rq);
}
```

#### 为什么切换容易

1. **接口统一**：所有调度算法都实现相同的接口，框架代码不需要修改。

2. **单点配置**：只需修改一处（`sched_class` 指针），即可切换整个系统的调度算法。

3. **运行时多态**：通过函数指针实现的多态机制，避免了大量的 if-else 或 switch-case 判断。

4. **模块化设计**：调度算法作为独立模块，可以单独开发和测试，不影响其他部分。

---

## 练习2：实现 Round Robin 调度算法

### 2.1 Lab5 与 Lab6 函数实现的对比分析

#### 对比函数：sched_class_proc_tick()

**Lab5 中不存在此函数**，时钟中断处理较为简单。

**Lab6 中的实现**（`kern/schedule/sched.c`）：
```c
void sched_class_proc_tick(struct proc_struct *proc)
{
    if (proc != idleproc)
    {
        sched_class->proc_tick(rq, proc);
    }
    else
    {
        proc->need_resched = 1;
    }
}
```

**为什么要做这个改动**：

1. **支持时间片轮转**：Lab6 的 RR 调度算法需要在每次时钟中断时更新进程的时间片，当时间片耗尽时触发调度。

2. **抽象时钟处理**：不同的调度算法对时钟中断有不同的处理需求：
   - RR 需要递减时间片
   - Stride 需要更新 stride 值
   - 通过 `proc_tick()` 函数指针，让具体算法自己处理

3. **处理空闲进程**：空闲进程总是愿意让出 CPU（`need_resched = 1`），保证有其他进程就绪时能立即切换。

**不做这个改动会出什么问题**：

- 进程的时间片不会更新，导致第一个进程会一直占用 CPU
- 即使有多个就绪进程，也无法实现轮转调度
- 系统退化为单进程运行模式，失去多任务并发能力

### 2.2 RR 调度算法实现详解

#### RR_init() - 初始化运行队列

```c
static void RR_init(struct run_queue *rq)
{
    list_init(&(rq->run_list));  // 初始化双向循环链表
    rq->proc_num = 0;             // 进程数量初始化为 0
}
```

**实现思路**：
- 使用双向循环链表管理就绪进程队列
- 初始化队列为空状态

**关键点**：
- `list_init()` 将链表的 prev 和 next 都指向自己，形成空的循环链表
- `proc_num` 用于快速查询队列中的进程数量

#### RR_enqueue() - 进程入队

```c
static void RR_enqueue(struct run_queue *rq, struct proc_struct *proc)
{
    assert(list_empty(&(proc->run_link)));
    list_add_before(&(rq->run_list), &(proc->run_link));  // 加入队尾
    if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
        proc->time_slice = rq->max_time_slice;  // 重置时间片
    }
    proc->rq = rq;    // 设置进程所属运行队列
    rq->proc_num++;   // 进程数量加 1
}
```

**实现思路**：
1. 检查进程的 `run_link` 是否为空（防止重复入队）
2. 使用 `list_add_before()` 将进程加入队尾（`run_list` 的前面即队尾）
3. 重置或初始化进程的时间片
4. 更新进程的运行队列指针和队列计数

**为什么选择 list_add_before()**：
- `run_list` 是队列的头节点（哨兵节点）
- `list_add_before(&run_list, &proc->run_link)` 将进程插入到头节点之前，即队尾
- 实现了 FIFO 队列：从队尾入队，从队头出队

**边界情况处理**：
- **空队列**：链表的循环结构自动处理，第一个进程会正确加入
- **时间片为 0**：重新分配完整的时间片，保证进程能运行
- **时间片过大**：限制为 `max_time_slice`，防止某个进程占用过长时间

#### RR_dequeue() - 进程出队

```c
static void RR_dequeue(struct run_queue *rq, struct proc_struct *proc)
{
    assert(!list_empty(&(proc->run_link)) && proc->rq == rq);
    list_del_init(&(proc->run_link));  // 从链表中删除并重新初始化
    rq->proc_num--;                     // 进程数量减 1
}
```

**实现思路**：
1. 检查进程确实在运行队列中
2. 使用 `list_del_init()` 从链表中删除并重新初始化节点
3. 更新队列计数

**为什么使用 list_del_init()**：
- `list_del()` 只删除节点，但节点的 prev 和 next 指针会变成野指针
- `list_del_init()` 在删除后将节点的 prev 和 next 都指向自己
- 这样可以使用 `list_empty()` 检查节点是否在链表中，避免重复操作

**边界情况处理**：
- **重复出队**：通过 `assert()` 检查，防止将不在队列中的进程出队
- **空队列出队**：通过检查 `list_empty()` 防止

#### RR_pick_next() - 选择下一个进程

```c
static struct proc_struct *RR_pick_next(struct run_queue *rq)
{
    if (list_empty(&(rq->run_list))) {
        return NULL;  // 队列为空，返回 NULL
    }
    list_entry_t *le = list_next(&(rq->run_list));  // 获取队头元素
    if (le != &(rq->run_list)) {
        return le2proc(le, run_link);  // 转换为进程指针
    }
    return NULL;
}
```

**实现思路**：
1. 检查队列是否为空
2. 获取队头元素（头节点的下一个节点）
3. 使用 `le2proc()` 宏将链表节点转换为进程结构体指针

**为什么选择队头**：
- 实现 FIFO 策略：最早入队的进程最先运行
- 与 `RR_enqueue()` 的队尾入队配合，实现时间片轮转

**le2proc() 宏的作用**：
```c
#define le2proc(le, member) to_struct((le), struct proc_struct, member)
```
- 已知结构体成员的地址（`le`）和成员名（`member`）
- 反向计算出结构体的起始地址
- 这是 Linux 内核中常用的技巧（`container_of` 宏）

**边界情况处理**：
- **空队列**：直接返回 NULL，调用者会选择 idleproc
- **双重检查**：既检查 `list_empty()` 又检查 `le != &run_list`，增强鲁棒性

#### RR_proc_tick() - 时钟中断处理

```c
static void RR_proc_tick(struct run_queue *rq, struct proc_struct *proc)
{
    if (proc->time_slice > 0) {
        proc->time_slice--;         // 时间片减 1
    }
    if (proc->time_slice == 0) {
        proc->need_resched = 1;     // 时间片耗尽，设置调度标志
    }
}
```

**实现思路**：
1. 每次时钟中断，当前进程的时间片减 1
2. 当时间片减到 0 时，设置 `need_resched` 标志
3. 标志会在中断返回前触发 `schedule()` 调用

**为什么需要设置 need_resched**：
1. **延迟调度**：不在中断处理函数中直接调度，而是设置标志延迟到中断返回前，避免嵌套中断的复杂性
2. **原子性保证**：在中断返回前的检查点调度，可以确保关中断，保证调度过程的原子性
3. **统一入口**：所有触发调度的场景（时间片耗尽、进程阻塞、主动让出 CPU）都通过这个标志，代码逻辑统一

**边界情况处理**：
- **时间片已经为 0**：只递减正数，避免负数
- **连续多次时钟中断**：已经设置的标志不会被重复设置

### 2.3 make grade 测试结果

```
priority:                (4.3s)
  -check result:                             OK
  -check output:                             OK
Total Score: 50/50
```

**完整测试日志**：

```
OpenSBI v0.4 (Jul  2 2019 11:53:53)
...
(THU.CST) os is loading ...

Special kernel symbols:
  entry  0xc020004a (virtual)
  etext  0xc020585e (virtual)
  edata  0xc02c2710 (virtual)
  end    0xc02c6bf0 (virtual)
Kernel executable memory footprint: 795KB
DTB Init
...
memory management: default_pmm_manager
...
check_alloc_page() succeeded!
check_pgdir() succeeded!
check_boot_pgdir() succeeded!
use SLOB allocator
kmalloc_init() succeeded!
check_vma_struct() succeeded!
check_vmm() succeeded.
sched class: RR_scheduler              ← 使用 RR 调度器
++ setup timer interrupts               ← 时钟中断初始化成功
kernel_execve: pid = 2, name = "priority".
set priority to 6
main: fork ok,now need to wait pids.
set priority to 1
set priority to 2
set priority to 3
set priority to 4
set priority to 5
100 ticks                               ← 时钟中断正常触发
100 ticks
child pid 7, acc 272000, time 2010
child pid 3, acc 296000, time 2020
child pid 4, acc 296000, time 2040
child pid 5, acc 296000, time 2050
child pid 6, acc 288000, time 2060
main: pid 0, acc 296000, time 2070
main: pid 4, acc 296000, time 2080
main: pid 5, acc 296000, time 2080
main: pid 6, acc 288000, time 2090
main: pid 7, acc 272000, time 2090
main: wait pids over
sched result: 1 1 1 1 1                 ← 各进程 CPU 时间比例接近 1:1
all user-mode processes have quit.      ← 所有用户进程正常退出
init check memory pass.                 ← 内存检查通过
kernel panic at kern/process/proc.c:544:
    initproc exit.
```

### 2.4 QEMU 中观察到的调度现象

#### 1. 进程轮转现象

从日志中可以观察到：
- 5 个子进程（优先级 1-5）在约 2000ms 内都完成了大量计算（acc 值）
- 各进程的 acc 值接近（272000-296000），说明获得了相近的 CPU 时间
- `sched result: 1 1 1 1 1` 表示各进程的 CPU 时间比例接近 1:1

#### 2. 时间片效果

- 每 100 个时钟中断（约 1000ms）输出一次 "100 ticks"
- 时间片设置为 5，意味着每个进程运行 5 个时钟中断后切换
- 2000ms 内大约发生了 20 次时间片轮转

#### 3. 公平性体现

- 即使优先级不同（priority 测试中设置了 1-6），RR 调度器仍然公平分配 CPU
- 这是因为当前 RR 实现不考虑优先级，严格按 FIFO 顺序调度

### 2.5 Round Robin 算法的优缺点分析

#### 优点

1. **公平性好**：
   - 所有进程按 FIFO 顺序获得相同的时间片
   - 不会有进程饿死（starvation）
   - 适合交互式系统，保证每个进程都能响应

2. **实现简单**：
   - 使用简单的链表即可实现
   - 入队、出队、选择下一个进程都是 O(1) 时间复杂度
   - 代码量少，易于理解和维护

3. **响应时间可预测**：
   - 如果有 n 个进程，每个时间片为 q，则最坏响应时间为 (n-1) * q
   - 适合实时性要求不高的系统

4. **无需进程优先级信息**：
   - 不需要复杂的优先级计算
   - 适合进程重要性相近的场景

#### 缺点

1. **不考虑进程特性**：
   - CPU 密集型和 I/O 密集型进程得到相同待遇
   - I/O 密集型进程的时间片大部分被浪费

2. **不支持优先级**：
   - 无法给重要进程更多 CPU 时间
   - 不适合需要差异化服务的场景

3. **平均等待时间可能较长**：
   - 对于短作业，可能需要等待多个长作业完成
   - 不如 SJF (Shortest Job First) 优化

4. **时间片大小难以确定**：
   - 时间片太大：退化为 FCFS，响应时间变长
   - 时间片太小：上下文切换开销大，CPU 利用率低

#### 时间片大小的调整策略

| 时间片大小 | 优点 | 缺点 | 适用场景 |
|-----------|------|------|---------|
| 很小 (1-2 ticks) | 响应时间短，交互性好 | 上下文切换频繁，开销大 | 交互式系统 |
| 中等 (5-10 ticks) | 平衡响应和开销 | 折中方案 | 通用系统 |
| 很大 (50+ ticks) | 上下文切换少，吞吐量高 | 响应时间长，交互性差 | 批处理系统 |

**优化建议**：
- 根据系统负载动态调整时间片
- 测量上下文切换开销占比，控制在 1-5% 以内
- 对于本实验，时间片为 5（约 50ms）是一个合理的中等值

#### 为什么需要在 RR_proc_tick 中设置 need_resched

1. **分离关注点**：
   - 时钟中断处理函数（`interrupt_handler`）只负责更新时间片
   - 进程调度逻辑（`schedule`）专注于选择和切换进程
   - 通过 `need_resched` 标志解耦两者

2. **安全性考虑**：
   - 在中断处理函数中直接调度可能导致：
     - 嵌套中断时的竞态条件
     - 死锁（如果调度函数内部需要获取锁）
   - 延迟到中断返回前调度，此时中断已经被正确处理，可以安全切换

3. **统一调度入口**：
   - 多种情况都可能触发调度：时间片耗尽、进程阻塞、主动让出、优先级变化
   - 通过统一的 `need_resched` 标志和 `schedule()` 入口，简化代码逻辑

4. **性能优化**：
   - 避免在每次时钟中断时都调用 `schedule()`
   - 只在真正需要调度时才调用，减少不必要的开销

### 2.6 拓展思考

#### 如何实现优先级 RR 调度

**方案 1：多级队列**

```c
#define PRIORITY_LEVELS 8

struct run_queue {
    list_entry_t run_lists[PRIORITY_LEVELS];  // 每个优先级一个队列
    unsigned int proc_num[PRIORITY_LEVELS];
    int max_time_slice;
};

// 入队时根据优先级选择队列
static void Priority_RR_enqueue(struct run_queue *rq, struct proc_struct *proc)
{
    int priority = proc->priority;
    assert(priority >= 0 && priority < PRIORITY_LEVELS);
    list_add_before(&(rq->run_lists[priority]), &(proc->run_link));
    proc->time_slice = rq->max_time_slice;
    rq->proc_num[priority]++;
}

// 选择时从高优先级队列开始
static struct proc_struct *Priority_RR_pick_next(struct run_queue *rq)
{
    for (int i = PRIORITY_LEVELS - 1; i >= 0; i--) {
        if (!list_empty(&(rq->run_lists[i]))) {
            list_entry_t *le = list_next(&(rq->run_lists[i]));
            return le2proc(le, run_link);
        }
    }
    return NULL;
}
```

**方案 2：动态时间片**

保持单队列，但根据优先级分配不同的时间片：

```c
static void Priority_RR_enqueue(struct run_queue *rq, struct proc_struct *proc)
{
    list_add_before(&(rq->run_list), &(proc->run_link));
    // 高优先级进程获得更长的时间片
    proc->time_slice = rq->max_time_slice * (proc->priority + 1);
    rq->proc_num++;
}
```

**代码修改要点**：
1. 修改 `proc_struct` 添加 `priority` 字段（已有 `lab6_priority`）
2. 修改 `enqueue()` 根据优先级插入或分配时间片
3. 修改 `pick_next()` 优先选择高优先级进程
4. 可选：添加优先级动态调整机制（防止低优先级进程饿死）

#### 当前实现是否支持多核调度

**当前不支持多核调度**，原因：

1. **单一运行队列**：
   - 只有一个全局 `run_queue`
   - 多个 CPU 会竞争同一个队列，需要大量同步

2. **单一 current 指针**：
   - 只有一个全局 `current` 指针
   - 无法表示多个 CPU 上同时运行的进程

3. **缺少 CPU 亲和性**：
   - 进程没有绑定到特定 CPU
   - 无法利用 CPU 缓存局部性

**改进方案**：

**方案 1：多队列（Per-CPU Queue）**

```c
struct run_queue {
    list_entry_t run_list;
    unsigned int proc_num;
    int max_time_slice;
    int cpu_id;                    // 新增：CPU ID
    spinlock_t lock;               // 新增：自旋锁保护队列
};

// 每个 CPU 一个运行队列和 current 指针
struct run_queue run_queues[NCPU];
struct proc_struct *currents[NCPU];

// 调度时只操作本 CPU 的队列
void schedule(void)
{
    int cpu_id = cpuid();  // 获取当前 CPU ID
    struct run_queue *rq = &run_queues[cpu_id];
    struct proc_struct *current = currents[cpu_id];
    
    acquire(&rq->lock);
    // 调度逻辑...
    release(&rq->lock);
}
```

**方案 2：负载均衡**

```c
// 定期进行负载均衡
void load_balance(void)
{
    int cpu_id = cpuid();
    struct run_queue *my_rq = &run_queues[cpu_id];
    
    // 找到负载最重的 CPU
    int max_load_cpu = find_max_load_cpu();
    if (max_load_cpu != cpu_id && 
        run_queues[max_load_cpu].proc_num > my_rq->proc_num + THRESHOLD) {
        // 迁移一些进程到本 CPU
        migrate_processes(max_load_cpu, cpu_id);
    }
}
```

**需要修改的部分**：
1. `proc_struct` 添加 `cpu_id` 字段记录进程当前所在 CPU
2. `run_queue` 变为数组，每个 CPU 一个
3. 添加自旋锁保护每个运行队列
4. 修改 `schedule()` 使用本 CPU 的运行队列
5. 添加负载均衡机制，定期迁移进程
6. 考虑 CPU 亲和性和缓存热度

---

## 实验总结

### 完成的工作

1. **成功实现了 RR 调度算法**：
   - 实现了 5 个核心函数，代码清晰、健壮
   - 通过了所有测试用例，得分 50/50

2. **深入理解了调度器框架**：
   - 理解了面向对象的设计思想在 C 语言中的应用
   - 掌握了通过函数指针实现多态的技巧
   - 理解了运行队列、时间片、进程切换等核心概念

3. **掌握了内核调试技能**：
   - 学会分析内核启动日志
   - 学会使用 `cprintf` 进行调试输出
   - 学会定位和修复编译错误和运行时错误

### 遇到的问题和解决

1. **问题**：初次编译时，进程控制块的 Lab6 字段未初始化，导致系统崩溃。
   - **解决**：在 `alloc_proc()` 中添加了所有 Lab6 字段的初始化代码。

2. **问题**：测试时系统在 100 ticks 后就 panic 退出，priority 测试无法完成。
   - **解决**：移除了 `print_ticks()` 中 DEBUG_GRADE 的 panic 代码。

3. **问题**：没有调用 `sched_class_proc_tick()`，进程无法轮转。
   - **解决**：在时钟中断处理函数中添加了调度器时钟处理的调用。

### 收获和体会

1. **设计模式的重要性**：良好的抽象设计（如 `sched_class`）使得代码易于扩展和维护。

2. **操作系统的复杂性**：看似简单的进程调度，实际涉及中断处理、上下文切换、内存管理等多个子系统的协调。

3. **细节决定成败**：一个字段未初始化、一个函数未调用，都可能导致系统完全无法运行。

4. **测试的价值**：自动化测试（make grade）能快速验证实现的正确性，是开发过程中的重要工具。

---

## 参考资料

1. uCore 实验指导书
2. 《操作系统概念》（恐龙书）第 6 章：CPU 调度
3. Linux 内核调度器源码（`kernel/sched/`）
4. RISC-V 特权架构规范

---

## 附录：关键代码清单

### A. 进程控制块初始化（proc.c）

```c
static struct proc_struct *alloc_proc(void)
{
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL)
    {
        proc->state = PROC_UNINIT;
        proc->pid = -1;
        proc->runs = 0;
        proc->kstack = 0;
        proc->need_resched = 0;
        proc->parent = NULL;
        proc->mm = NULL;
        memset(&(proc->context), 0, sizeof(struct context));
        proc->tf = NULL;
        proc->pgdir = boot_pgdir_pa;
        proc->flags = 0;
        memset(proc->name, 0, PROC_NAME_LEN + 1);
        
        proc->wait_state = 0;
        proc->cptr = proc->yptr = proc->optr = NULL;
        
        // LAB6: 初始化调度相关字段
        proc->rq = NULL;
        list_init(&(proc->run_link));
        proc->time_slice = 0;
        proc->lab6_run_pool.left = proc->lab6_run_pool.right = 
                                    proc->lab6_run_pool.parent = NULL;
        proc->lab6_stride = 0;
        proc->lab6_priority = 0;
    }
    return proc;
}
```

### B. RR 调度算法完整实现（default_sched.c）

```c
#include <defs.h>
#include <list.h>
#include <proc.h>
#include <assert.h>
#include <default_sched.h>

static void RR_init(struct run_queue *rq)
{
    list_init(&(rq->run_list));
    rq->proc_num = 0;
}

static void RR_enqueue(struct run_queue *rq, struct proc_struct *proc)
{
    assert(list_empty(&(proc->run_link)));
    list_add_before(&(rq->run_list), &(proc->run_link));
    if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
        proc->time_slice = rq->max_time_slice;
    }
    proc->rq = rq;
    rq->proc_num++;
}

static void RR_dequeue(struct run_queue *rq, struct proc_struct *proc)
{
    assert(!list_empty(&(proc->run_link)) && proc->rq == rq);
    list_del_init(&(proc->run_link));
    rq->proc_num--;
}

static struct proc_struct *RR_pick_next(struct run_queue *rq)
{
    if (list_empty(&(rq->run_list))) {
        return NULL;
    }
    list_entry_t *le = list_next(&(rq->run_list));
    if (le != &(rq->run_list)) {
        return le2proc(le, run_link);
    }
    return NULL;
}

static void RR_proc_tick(struct run_queue *rq, struct proc_struct *proc)
{
    if (proc->time_slice > 0) {
        proc->time_slice--;
    }
    if (proc->time_slice == 0) {
        proc->need_resched = 1;
    }
}

struct sched_class default_sched_class = {
    .name = "RR_scheduler",
    .init = RR_init,
    .enqueue = RR_enqueue,
    .dequeue = RR_dequeue,
    .pick_next = RR_pick_next,
    .proc_tick = RR_proc_tick,
};
```

### C. 时钟中断处理（trap.c）

```c
case IRQ_S_TIMER:
    clock_set_next_event();
    ticks++;
    if (ticks % TICK_NUM == 0) {
        print_ticks();
    }
    // LAB6: 调用调度器的时钟处理函数
    sched_class_proc_tick(current);
    break;
```

---

**实验完成时间**：2026年1月6日  
**测试得分**：50/50  
**实验状态**：全部通过 ✅
