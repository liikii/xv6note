## kinit1
前4mb空闲页
1. 临时页表：只管 4MB，勉强能跑 main。
2. kinit1：把这 4MB 里没被内核代码占用的部分变成“启动资金”。
3. setupkvm：从这 4MB 的“启动资金”里拿出 228KB，画出一张覆盖 224MB 的全图。
4. switchkvm：戴上这副“全图”眼镜。


## kvmalloc  setupkvm
把kmap的段分成4kb,4kb的块，写每个页表。 


kmap[] = {
 { (void*)KERNBASE, 0,             EXTMEM,    PTE_W}, // I/O space
 { (void*)KERNLINK, V2P(KERNLINK), V2P(data), 0},     // kern text+rodata
 { (void*)data,     V2P(data),     PHYSTOP,   PTE_W}, // kern data+memory
 { (void*)DEVSPACE, DEVSPACE,      0,         PTE_W}, // more devices
};

