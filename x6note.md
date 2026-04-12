## kinit1
前4mb空闲页
1. 临时页表：只管 4MB，勉强能跑 main。
2. kinit1：把这 4MB 里没被内核代码占用的部分变成“启动资金”。
3. setupkvm：从这 4MB 的“启动资金”里拿出 228KB，画出一张覆盖 224MB 的全图。
4. switchkvm：戴上这副“全图”眼镜。


## kvmalloc  setupkvm
把kmap的段分成4kb,4kb的块，写每个页表。 

```c
kmap[] = {
 { (void*)KERNBASE, 0,             EXTMEM,    PTE_W}, // I/O space
 { (void*)KERNLINK, V2P(KERNLINK), V2P(data), 0},     // kern text+rodata
 { (void*)data,     V2P(data),     PHYSTOP,   PTE_W}, // kern data+memory
 { (void*)DEVSPACE, DEVSPACE,      0,         PTE_W}, // more devices
};
```


## mp.c  mp mpconf


```text
[ 低地址 BIOS 区域 (通常在 0xF0000 附近) ]
+--------------------------+

|      struct mp           | <--- 浮动指针结构 (Floating Pointer)
|  signature: "_MP_"       |
|  physaddr: 0x000F1234 ---|---- 存储了 mpconf 的物理地址
+--------------------------+    |

                                |
      (中间隔着大量的其他数据)      |
                                |
[ 物理地址 0x000F1234 ]         |
+--------------------------+ <--- 指向这里

|    struct mpconf         | <--- 配置表头 (Table Header)
|  signature: "PCMP"       |
+--------------------------+

|  Entries (CPU/IOAPIC...) | <--- 紧跟在 mpconf 后面的条目
+--------------------------+



内存地址低 -> 高
+-----------------------+ <--- conf (struct mpconf 指针指向这里)

|  PCMP (Signature)     | ^
|  Length (总长度)       | |
|  ...                  | | struct mpconf (表头)
|  Entry Count (条目数)  | |
|  LAPIC Address        | |
+-----------------------+ v

|  Type (MPPROC)        | ^
|  Processor Data...    | | 第一条数据 (struct mpproc)
+-----------------------+ v

|  Type (MPPROC)        | ^
|  Processor Data...    | | 第二条数据 (struct mpproc)
+-----------------------+ v

|  Type (MPIOAPIC)      | ^
|  IOAPIC Data...       | | 第三条数据 (struct mpioapic)
+-----------------------+ v

|  ...                  |



地址低 -> 高
[ Type=0 ] | [ mpproc 其余字段... ]  <-- 这是一个 CPU (20 字节)
[ Type=0 ] | [ mpproc 其余字段... ]  <-- 这是另一个 CPU (20 字节)
[ Type=2 ] | [ mpioapic 其余字段... ] <-- 这是一个 IO APIC (12 字节)
[ Type=1 ] | [ 总线数据... ]         <-- 这是一个 BUS (8 字节)
```


mpinit 的核心本质就是“信息普查”与“建立索引”。
它并不负责真正激活硬件（那是后续函数的事），它只是把 BIOS 留在内存里的那些零散的硬件参数“捞”出来，存到内核容易访问的全局变量里。


开启apic
outb(0x22, 0x70);   // Select IMCR
outb(0x23, inb(0x23) | 1);  // Mask external interrupts.


## lapic cpu的中断设置



```c
static void
lapicw(int index, int value)
{
  lapic[index] = value; // 1. 向特定寄存器写入控制值
  lapic[ID];            // 2. 这里的“读”操作是为了“同步”
}

```
读取 lapic[ID]：这行代码看起来没用，但非常关键。因为 LAPIC 寄存器映射在内存上，由于 CPU 的缓存或指令重排，写入操作可能还没真正到达硬件。通过立即执行一次读取（通常读 ID 寄存器，因为它最安全且无副作用），强制 CPU 完成之前的写入指令，确保硬件已经接收到了指令。


往 lapic[index] 写入的值，其格式取决于你具体在操作哪一个寄存器。它们大多数是 32 位的位图（Bitmask），每一位（bit）或每一组位都代表一个特定的硬件开关。
虽然每个寄存器不同，但最核心的格式主要分为以下三类：
## 1. 简单的数值/ID 格式
这类寄存器比较直接，直接存入一个整数。

* 示例：lapic[ID]（虽然通常是只读或由 BIOS 设置）。
* 用法：写入或读取一个 8 位的 APIC ID，用于标识当前 CPU。

## 2. 带有“掩码”和“模式”的配置格式 (LVT 寄存器)
这是最常见的格式，用于 定时器 (Timer) 或 热传感器 (Thermal)。

* 格式拆解：
* 0-7 位：中断向量号 (Vector)。即触发中断时，去执行中断向量表里的哪一个程序。
   * 8-10 位：交付模式 (Delivery Mode)。
   * 12 位：交付状态 (Delivery Status)。0 表示空闲，1 表示正在发送。
   * 16 位：屏蔽位 (Mask)。写 1 则关闭该中断，写 0 则开启。
* xv6 例子：开启定时器中断。

lapicw(TIMER, PERIODIC | (T_IRQ0 + IRQ_TIMER)); // PERIODIC 是模式位，T_IRQ0... 是向量号位


## 3. 最复杂的 ICR (中断控制寄存器) 格式
这是 CPU 之间发消息（IPI） 的专用格式，通常是两个 32 位寄存器配合使用。

* ICRHI (高32位)：包含 目标 CPU 的 APIC ID。
* ICRLO (低32位)：包含 发送指令：
* Level：是电平触发还是边沿触发。
   * Destination Shorthand：发给谁？（发给自己、发给除了自己以外的所有人、还是发给特定的人）。
   * Delivery Mode：是普通中断，还是 INIT（强制初始化信号），还是 STARTUP（唤醒信号）。

## 总结
你可以把这些 value 想象成一排 “拨码开关”：

* 有的开关管频率（定时器数值）。
* 有的开关管地址（向量号）。
* 有的开关管生死（Mask 屏蔽位）。

为什么要这么复杂？
因为硬件引脚有限，工程师必须在一个 32 位的整数里塞进尽可能多的控制信息。在 xv6 的 lapic.c 中，你会看到很多类似 0x00000400（设置模式）或 0x00010000（屏蔽中断）的十六进制常数，它们就是在操作这些特定的“开关位”。
你想深入看看 ICR 寄存器 是如何组合成一个 value 来唤醒一个“沉睡”的核心的吗？



lapic只设置了timer和error中断处理。 
外部设备中断不归 LAPIC 负责设置，而归 IOAPIC 负责。


1. Timer (用于调度)
2. Error (用于容错)
3. Spurious (用于稳定)


无论中断是从 IOAPIC（外部硬件）发来的，还是从 LAPIC（内部定时器/错误）产生的，它们最终都化作一个向量号（Vector Number），去 IDT 里寻找处理函数。


