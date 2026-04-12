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


---

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
1. 简单的数值/ID 格式
这类寄存器比较直接，直接存入一个整数。

* 示例：lapic[ID]（虽然通常是只读或由 BIOS 设置）。
* 用法：写入或读取一个 8 位的 APIC ID，用于标识当前 CPU。

2. 带有“掩码”和“模式”的配置格式 (LVT 寄存器)
这是最常见的格式，用于 定时器 (Timer) 或 热传感器 (Thermal)。

* 格式拆解：
* 0-7 位：中断向量号 (Vector)。即触发中断时，去执行中断向量表里的哪一个程序。
   * 8-10 位：交付模式 (Delivery Mode)。
   * 12 位：交付状态 (Delivery Status)。0 表示空闲，1 表示正在发送。
   * 16 位：屏蔽位 (Mask)。写 1 则关闭该中断，写 0 则开启。
* xv6 例子：开启定时器中断。

lapicw(TIMER, PERIODIC | (T_IRQ0 + IRQ_TIMER)); // PERIODIC 是模式位，T_IRQ0... 是向量号位


3. 最复杂的 ICR (中断控制寄存器) 格式
这是 CPU 之间发消息（IPI） 的专用格式，通常是两个 32 位寄存器配合使用。

* ICRHI (高32位)：包含 目标 CPU 的 APIC ID。
* ICRLO (低32位)：包含 发送指令：
* Level：是电平触发还是边沿触发。
   * Destination Shorthand：发给谁？（发给自己、发给除了自己以外的所有人、还是发给特定的人）。
   * Delivery Mode：是普通中断，还是 INIT（强制初始化信号），还是 STARTUP（唤醒信号）。

总结
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

---

## gdt

```c
seginit(void)
{
  struct cpu *c;

  // Map "logical" addresses to virtual addresses using identity map.
  // Cannot share a CODE descriptor for both kernel and user
  // because it would have to have DPL_USR, but the CPU forbids
  // an interrupt from CPL=0 to DPL=3.
  c = &cpus[cpuid()];
  c->gdt[SEG_KCODE] = SEG(STA_X|STA_R, 0, 0xffffffff, 0);
  c->gdt[SEG_KDATA] = SEG(STA_W, 0, 0xffffffff, 0);
  c->gdt[SEG_UCODE] = SEG(STA_X|STA_R, 0, 0xffffffff, DPL_USER);
  c->gdt[SEG_UDATA] = SEG(STA_W, 0, 0xffffffff, DPL_USER);
  lgdt(c->gdt, sizeof(c->gdt));
}
```
这段代码的作用是为当前 CPU 初始化 GDT (Global Descriptor Table，全局描述符表)。
在 x86 架构中，即使 xv6 主要使用页表（Paging）来管理内存，硬件依然要求必须设置 GDT 来定义代码和数据的访问权限（特权级）。
1. 核心逻辑：分而治之
seginit 在多核环境下会被每个 CPU 核心各调用一次。

c = &cpus[cpuid()];


* cpuid()：通过读取我们之前提到的 LAPIC ID，确定当前运行的是哪一个 CPU。
* c->gdt：每个 struct cpu 内部都有自己独立的 gdt 数组。这意味着每个核心都有自己的权限表。

2. 详解四个段描述符 (The Segments)
xv6 使用的是“扁平模型”（Flat Model），即所有段的基地址都是 0，范围都是 4GB。它们唯一的区别是权限。

* SEG_KCODE (内核代码段)
* STA_X|STA_R：可执行、可读。
   * DPL=0：最高权限（Kernel 级别）。
* SEG_KDATA (内核数据段)
* STA_W：可写。
   * DPL=0：最高权限。用于内核堆栈和全局变量。
* SEG_UCODE (用户代码段)
* DPL_USER (即 3)：最低权限。用户程序跑在这里。
* SEG_UDATA (用户数据段)
* DPL_USER：最低权限。用户程序的堆栈和数据。

3. 深入理解注释里的“坑”
代码里的这段注释非常关键：

Cannot share a CODE descriptor for both kernel and user because... the CPU forbids an interrupt from CPL=0 to DPL=3.

通俗解释：

* CPL (Current Privilege Level)：当前 CPU 正在跑的级别（内核是 0，用户是 3）。
* DPL (Descriptor Privilege Level)：段描述符规定的级别。
* 硬件规则：如果内核（CPL=0）正在运行，此时发生了一个中断，CPU 要求目标代码段的权限（DPL）必须等于或高于当前级别。
* 结论：如果内核共用一个 DPL=3 的段，当中断发生时，CPU 会认为你要“跳入一个低权限区域”，这在硬件层面是不安全的，会导致 CPU 直接崩溃（General Protection Fault）。所以，必须为内核专门准备一个 DPL=0 的段。

4. 激活 GDT

lgdt(c->gdt, sizeof(c->gdt));

这是最后一步：通过汇编指令 lgdt 将这个数组的地址和大小加载到 CPU 内部的 GDTR 寄存器中。从这一刻起，当前 CPU 核心就知道如何区分内核态和用户态了。
总结
seginit 就像是给每个 CPU 核心下发了一本“身份证明”：

   1. 它定义了什么是高权限（内核），什么是低权限（用户）。
   2. 它通过 SEG 宏将基址 0 和长度 4GB 填入，确保页表（Paging）能接管后续的地址转换。
   3. 它解决了中断跳转时的权限校验问题。

你想看看 SEG 宏 内部是如何通过位运算把这些权限拼接成硬件要求的 8 字节格式的吗？


---

## picinit 关闭旧pic

mpinit 里的操作是“改道”。
picinit 里的操作是“停机”。


---

## ioapic 初始化


mp.c里不是从mp config里取了  ioapic = (struct mpioapic*)p;吗， 为什么这里ioapic = (volatile struct ioapic*)IOAPIC; 直接把地址写做常量了
mp.c 里的代码是通用的“普查”，而 ioapic.c 里的代码为了简单而“偷了懒”。

想象 IOAPIC 是一个有几十个小抽屉的大柜子，但你面前只有两个设备：
1. 一个键盘（reg）：让你输入抽屉编号。
2. 一个开口（data）：当你输入编号后，那个抽屉的内容就会出现在这个开口里。


* 想读 10 号抽屉？
先往 reg 写 10，然后从 data 读值。
* 想往 20 号抽屉放东西？
先往 reg 写 20，然后往 data 写值。


---

## 原版xv6没有遍历pci 
既然原版 xv6 没有 PCI 配置空间，那它怎么和硬件（磁盘、网卡、VGA）交互？
答案很简单：它不和 PCI 交互。xv6 采用的是 “静态直连硬件” 的方式，直接写死硬件地址，跳过 PCI 枚举。


---


## pci 和ioapic 解析关系不大

**IOAPIC 本身不是 PCI 设备，也不由 PCI 枚举发现；
但 IOAPIC 的核心工作——**给 PCI 设备发中断**——高度依赖 PCI 配置空间里的信息。**
它们是**“中断路由搭档”**的关系。
1. 它们怎么被发现？（完全两条路）
* IOAPIC 怎么来？
- 从 **MP 配置表（mpinit 扫的那个）** 里读出来
- 或从 ACPI 表里读
- **和 PCI 一毛钱关系没有**

* PCI 设备怎么来？
- 用 **0xCF8 / 0xCFC 端口枚举**
- 扫 bus/dev/func
- **和 MP 表/IOAPIC 无关**

“PCI 枚举 + 读取 IRQ + 配置 IOAPIC”

