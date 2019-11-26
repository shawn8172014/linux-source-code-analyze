## 中断处理 - 上半部

由于 `APIC中断控制器` 有点小复杂，所以本文主要通过 `8259A中断控制器` 来介绍Linux对中断的处理过程。

### 中断处理相关结构

前面说过，`8259A中断控制器` 由两片 8259A 风格的外部芯片以 `级联` 的方式连接在一起，每个芯片可处理多达 8 个不同的 IRQ（中断请求），所以可用 IRQ 线的个数达到 15 个。如下图：

![8259A](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/8259A.png)

在内核中每条IRQ线由结构体 `irq_desc_t` 来描述，`irq_desc_t` 定义如下：
```c
typedef struct {
    unsigned int status;        /* IRQ status */
    hw_irq_controller *handler;
    struct irqaction *action;   /* IRQ action list */
    unsigned int depth;         /* nested irq disables */
    spinlock_t lock;
} irq_desc_t;
```
下面介绍一下 `irq_desc_t` 结构各个字段的作用：
* `status`: IRQ线的状态。
* `handler`: 类型为 `hw_interrupt_type` 结构，表示IRQ线对应的硬件相关处理函数，比如 `8259A中断控制器` 接收到一个中断信号时，需要发送一个确认信号才会继续接收中断信号的，发送确认信号的函数就是 `hw_interrupt_type` 中的 `ack` 函数。
* `action`: 类型为 `irqaction` 结构，中断信号的处理入口。由于一条IRQ线可以被多个硬件共享，所以 `action` 是一个链表，每个 `action` 代表一个硬件的中断处理入口。
* `depth`: 防止多次开启和关闭IRQ线。
* `lock`: 防止多核CPU同时对IRQ进行操作的自旋锁。

`hw_interrupt_type` 这个结构与硬件相关，这里就不作介绍了，我们来看看 `irqaction` 这个结构：
```c
struct irqaction {
    void (*handler)(int, void *, struct pt_regs *);
    unsigned long flags;
    unsigned long mask;
    const char *name;
    void *dev_id;
    struct irqaction *next;
};
```
下面说说 `irqaction` 结构各个字段的作用：
* `handler`: 中断处理的入口函数，`handler` 的第一个参数是中断号，第二个参数是设备对应的ID，第三个参数是中断发生时由内核保存的各个寄存器的值。
* `flags`: 标志位，用于表示 `irqaction` 的一些行为，例如是否能够与其他硬件共享IRQ线。
* `name`: 用于保存中断处理的名字。
* `dev_id`: 设备ID。
* `next`: 每个硬件的中断处理入口对应一个 `irqaction` 结构，由于多个硬件可以共享同一条IRQ线，所以这里通过 `next` 字段来连接不同的硬件中断处理入口。

`irq_desc_t` 结构关系如下图：

![irq_desc_t](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/irq_desc_t.jpg)

### 注册中断处理入口
在内核中，可以通过 `setup_irq()` 函数来注册一个中断处理入口。`setup_irq()` 函数代码如下：
```c
int setup_irq(unsigned int irq, struct irqaction * new)
{
    int shared = 0;
    unsigned long flags;
    struct irqaction *old, **p;
    irq_desc_t *desc = irq_desc + irq;
    ...
    spin_lock_irqsave(&desc->lock,flags);
    p = &desc->action;
    if ((old = *p) != NULL) {
        if (!(old->flags & new->flags & SA_SHIRQ)) {
            spin_unlock_irqrestore(&desc->lock,flags);
            return -EBUSY;
        }

        do {
            p = &old->next;
            old = *p;
        } while (old);
        shared = 1;
    }

    *p = new;

    if (!shared) {
        desc->depth = 0;
        desc->status &= ~(IRQ_DISABLED | IRQ_AUTODETECT | IRQ_WAITING);
        desc->handler->startup(irq);
    }
    spin_unlock_irqrestore(&desc->lock,flags);

    register_irq_proc(irq); // 注册proc文件系统
    return 0;
}
```
`setup_irq()` 函数比较简单，就是通过 `irq` 号来查找对应的 `irq_desc_t` 结构，并把新的 `irqaction` 连接到 `irq_desc_t` 结构的 `action` 链表中。要注意的是，如果设备不支持共享IRQ线（也即是 `flags` 字段没有设置 `SA_SHIRQ` 标志），那么就返回 `EBUSY` 错误。

我们看看 `时钟中断处理入口` 的注册实例：
```c
static struct irqaction irq0  = { timer_interrupt, SA_INTERRUPT, 0, "timer", NULL, NULL};

void __init time_init(void)
{
    ...
    setup_irq(0, &irq0);
}
```
可以看到，时钟中断处理入口的IRQ号为0，并且处理函数为 `timer_interrupt()`。