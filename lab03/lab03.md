IA-32存储管理和保护
========

###实验过程

首先看IA-32的段式内存管理转换。通过查阅相关文档，可以看出，段式内存虚拟地址转化成线性地址的过程是这样的：首先通过对地址的解码得知所需的段寄存器是CS DS 还是SS。然后通过段寄存器在GDT或LDT里寻找到对应的段描述符，并提取相应的CPL DPL RPL进行权限判断以实现存储保护，以及将段的大小与虚拟地址做相应比较，若权限和其他条件判断符合要求，便通过段描述符中的段基址+虚拟地址得到对应的线性地址。于是segment_translation代码如下（包含条件编译的调试代码）：

```c
bool segment_translation(uint8_t access_target, uint32_t vaddr, uint32_t *laddr) {
```

然后是将线性地址转化成物理地址。由于本实验默认为4K页结构，根据文档说明（Intel 64 and IA-32 Architectures Software Developer's Manual: 4-10至4-16 Vol. 3A），可知线性地址的31:22位用于确定目录地址，21:12位用于确定对应页表，11:0位为对应物理页框中的偏移量。页目录地址的确定方式为：

* 31:12 位来自 CR3
* 11:2  位来自线性地址31:22位
* 1:0   位为0

PTE的地址确定方式：

* 31:12 位来自 PDE 的页框
* 11:2  位来自线性地址21:12位
* 1:0   位为0

最终物理地址的确定方式：

* 31:12 位来自 PTE 的页框
* 11:0  位来自线性地址11:0位

页转换时需要判断页项的p r/w u/s位，判断是否符合要求，然后再通过上述过程进行转化：

```c
bool page_translation(uint8_t rw_type, uint32_t laddr, uint32_t *paddr) {
```

至于load_selector，只需判断cpl是否小于3，若是则将selector加载到对应的段寄存器中。如下：

```c
bool load_selector(struct SegmentSelector *seg_reg, uint16_t selector) {
```

故实现最终的float版mysum如下：

```c
float mysum(int nr_num, ...) {
    int i;
    double sum=0;
    void* ap;
    double* fap;
    ap=(void*)&nr_num;
    ap+=sizeof(int);
    for(i=0; i<nr_num; i++) {
        fap=(double*)ap;
        sum+=*fap;
        ap+=sizeof(double);
    }
    return (float)sum;
}
```

由于助教君忘记在切换内核态至用户态的过程中变更DS，U/S位处理失误等原因，最终结果如下（注释在x86.h中的define debug）：

```
justin@debian:~/arch-lab3$ make
```

###思考题

采用段式内存管理的缺点：易造成内存碎片化，浪费空间。

采用页式内存管理的优点：可以让每个应用程序均享有很大的虚拟地址空间，同时还能保证内存碎片化现象不会十分严重。

如果采用单级页表，那么每一个进程均需把自己的完整页表放在内存中。如果采用较小的页大小，则每个页表的大小会比较大；若采用更大的页，那么会导致严重的内存碎片化现象，并导致更大的缺页代价。当地址空间增大时，这种现象将会更加严重。若采用多级页表，可以在线程运行的过程中填补页，代价不会很高，并且总体而言可以减少总的页表大小。

###一些想法

助教君，要仔细看看文档啊……U/S位为0时是SUPERVISOR，1时是USER这种事情……