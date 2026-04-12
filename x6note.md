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

