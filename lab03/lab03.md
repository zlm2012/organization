IA-32存储管理和保护
========

###实验过程

首先看IA-32的段式内存管理转换。通过查阅相关文档，可以看出，段式内存虚拟地址转化成线性地址的过程是这样的：首先通过对地址的解码得知所需的段寄存器是CS DS 还是SS。然后通过段寄存器在GDT或LDT里寻找到对应的段描述符，并提取相应的CPL DPL RPL进行权限判断以实现存储保护，以及将段的大小与虚拟地址做相应比较，若权限和其他条件判断符合要求，便通过段描述符中的段基址+虚拟地址得到对应的线性地址。于是segment_translation代码如下（包含条件编译的调试代码）：

```c
bool segment_translation(uint8_t access_target, uint32_t vaddr, uint32_t *laddr) {	/* implement this function to perform segment translation */	uint32_t rpl,cpl,dpl, baseaddr, seglimit, granularity;	cpl=read_cpl();	switch(access_target) {		case('c'):			rpl=cs.request_privilege_level;			dpl=gdt[cs.descriptor_index].privilege_level;			#ifdef debug			printf("descindex:%d\n",cs.descriptor_index);			#endif			seglimit=(gdt[cs.descriptor_index].limit_19_16<<16)+gdt[cs.descriptor_index].limit_15_0;			granularity=gdt[cs.descriptor_index].granularity;			baseaddr=(gdt[cs.descriptor_index].base_31_24<<24)+(gdt[cs.descriptor_index].base_23_16<<16)+gdt[cs.descriptor_index].base_15_0;			break;		case('d'):			rpl=ds.request_privilege_level;			dpl=gdt[ds.descriptor_index].privilege_level;			#ifdef debug			printf("descindex:%d\n",ds.descriptor_index);			#endif			seglimit=(gdt[ds.descriptor_index].limit_19_16<<16)+gdt[ds.descriptor_index].limit_15_0;			granularity=gdt[ds.descriptor_index].granularity;			baseaddr=(gdt[ds.descriptor_index].base_31_24<<24)+(gdt[ds.descriptor_index].base_23_16<<16)+gdt[ds.descriptor_index].base_15_0;			break;		case('s'):			rpl=ss.request_privilege_level;			dpl=gdt[ss.descriptor_index].privilege_level;			#ifdef debug			printf("descindex:%d\n",ss.descriptor_index);			#endif			seglimit=(gdt[ss.descriptor_index].limit_19_16<<16)+gdt[ss.descriptor_index].limit_15_0;			granularity=gdt[ss.descriptor_index].granularity;			baseaddr=(gdt[ss.descriptor_index].base_31_24<<24)+(gdt[ss.descriptor_index].base_23_16<<16)+gdt[ss.descriptor_index].base_15_0;			break;		case('*'):case('#'):return true; break;		default:return false;		}	if(granularity)seglimit<<=12;	if(vaddr>=seglimit)return false;	if(dpl>=rpl && dpl>=cpl){	*laddr=baseaddr+vaddr;	#ifdef debug	printf("%x\n",*laddr);	#endif	return true;	}	#ifdef debug	else if(dpl<rpl)printf("rpl=%d, dpl=%d\n",rpl, dpl);	else if(dpl<cpl)printf("cpl=%d, dpl=%d\n",cpl, dpl);	#endif	return false;}
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
bool page_translation(uint8_t rw_type, uint32_t laddr, uint32_t *paddr) {	/* implement this function to perform page translation, as well as page-level protection */	uint32_t segdesc,cpl;	bool write;	cpl=read_cpl();	switch(rw_type){		case('r'):			write=false;			break;		case('w'):			write=true;			break;		case('l'):			segdesc=laddr&0xffff;			load_selector(&ds,segdesc);			return true;			break;		case('*'):			do_irq();			return true;			break;		case('#'):			if(cpl==0){				do_irq_finish();				return true;			}else return false;			break;		default: printf("default...\n");return false;	}	struct PageDirectoryEntry* pde=(struct PageDirectoryEntry*)((cr3.page_directory_base<<12)+((laddr>>22)<<2));	if(!pde->present){		#ifdef debug		printf("present_d\n");		#endif		return false;	}	if(pde->read_write==0&&write){		#ifdef debug		printf("read_write_d\n");		#endif		return false;	}	if(cpl==3&&pde->user_supervisor==0){		#ifdef debug		printf("u/s_d\n");		#endif		return false;	}	struct PageTableEntry* pte=(struct PageTableEntry*)(pde->page_frame<<12)+(((laddr&0x003ff000)>>12)<<2);	if(!pte->present){		#ifdef debug		printf("present\n");		#endif		return false;	}	if(pte->read_write==0&&write==true){		#ifdef debug		printf("rw\n");		#endif		return false;	}	if(cpl==3&&pte->user_supervisor==0){		#ifdef debug		printf("u/s\n");		#endif		return false;	}	*paddr=(pte->page_frame<<12)+(laddr&0x00000fff);	return true;}
```

至于load_selector，只需判断cpl是否小于3，若是则将selector加载到对应的段寄存器中。如下：

```c
bool load_selector(struct SegmentSelector *seg_reg, uint16_t selector) {	/* implement this function to perform segment-level protection */	/*uint32_t rpl,cpl,dpl;	cpl=read_cpl();	dpl=gdt[(selector&0x1fff)].privilege_level;	rpl=*/	#ifdef debug	printf("load_selector\n");	#endif	if(read_cpl()>2)return false;		seg_reg->request_privilege_level=0x00000003&selector;	seg_reg->table_indicator=(0x00000004&selector)>>2;	seg_reg->descriptor_index=(0xfffffff8&selector)>>3;	return true;}
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
justin@debian:~/arch-lab3$ makegcc -m32 -O2 -Wall -Werror -fno-strict-aliasing   -c -o paging.o paging.cgcc -m32 -O2 -Wall -Werror -fno-strict-aliasing   -c -o irq.o irq.cgcc -m32 -O2 -Wall -Werror -fno-strict-aliasing   -c -o main.o main.cgcc -m32 -O2 -Wall -Werror -fno-strict-aliasing   -c -o segmentation.o segmentation.cgcc -o x86mm -m32 -O2 -Wall -Werror -fno-strict-aliasing ./paging.o ./irq.o ./main.o ./segmentation.ojustin@debian:~/arch-lab3$ ./x86mm ds rpl=0, index=2start kernel initialization...0xc0100000 --> 0x004000000xc0100004 --> 0x004000040xc0100008 --> 0x004000080xc0101000 --> 0x004040000xc0101004 --> 0x004040040x00000000 --> 0x000000000xc0102000 --> 0x004080000xc0102004 --> 0x004080040xc0110000 --> 0x004400000xc0111000 --> 0x00444000start user code...# Segmentation fault# Segmentation fault# Segmentation fault# Segmentation fault# Segmentation fault# Segmentation fault# Segmentation fault# Segmentation fault# Segmentation fault# Segmentation fault# Segmentation faultan interrupt comes!0xc0100000 --> 0x004000000xc0104321 --> 0x004103210x08600000 --> 0x03000000# Segmentation fault# Segmentation faultchange DS to 0x00080x08000000 --> 0x024000000x08800000 --> 0x02c00000# Segmentation faultfinish handling interrupt!# Segmentation fault# Segmentation fault# Segmentation faultsimulation end!
```

###思考题

采用段式内存管理的缺点：易造成内存碎片化，浪费空间。

采用页式内存管理的优点：可以让每个应用程序均享有很大的虚拟地址空间，同时还能保证内存碎片化现象不会十分严重。

如果采用单级页表，那么每一个进程均需把自己的完整页表放在内存中。如果采用较小的页大小，则每个页表的大小会比较大；若采用更大的页，那么会导致严重的内存碎片化现象，并导致更大的缺页代价。当地址空间增大时，这种现象将会更加严重。若采用多级页表，可以在线程运行的过程中填补页，代价不会很高，并且总体而言可以减少总的页表大小。

###一些想法

助教君，要仔细看看文档啊……U/S位为0时是SUPERVISOR，1时是USER这种事情……