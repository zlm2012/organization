printf与函数调用
========

###实验过程

首先，通过对讲义中的例子的思考，可以得知，IA32 下的 Calling Convention 是通过压栈进行的，即：调用前将参数由后至前压到栈帧的较高位置中（即第一个参数放至 %esp 位置），然后调用，而后被调用的过程便通过对调用栈的访问来获取所需的参数。这个过程中包含两个重要信息：一是所有的参数都是按顺序连续存放的，二是所有的参数都在内存中。故由此可实现整数版本的 mysum：

```c
int mysum(int nr_num, ...) {
    int sum=0, *ap, i;
    ap=&nr_num;
    ap++;
    for (i=0; i<nr_num; i++)
        sum+=*ap++;
    return sum;
}
```

此版本在如下main的调用后运行正常：

```c
int main(int argc, char** argv) {
    printf("%d,%d,%d\n", mysum(2, 1, 2), mysum(3, 2, 1, 3), mysum(2, 1, 2, 4, 5));
    return 0;
}
```

故继续挑战选做内容。选做是将int变化成float。最开始我按float去编写，但总是无法完成。通过看返汇编结果发现每次压入栈中的浮点数全部都是8字节的，查看保存浮点数的堆可以发现之中保存的全部是double型。为了验证这一点，使用标准库stdarg.h做一个简单的float版mysum，编译时的提示信息验证了我的猜想：

```
mysum.c: In function ‘mysum’:
mysum.c:9:14: warning: ‘float’ is promoted to ‘double’ when passed through ‘...’ [enabled by default]
mysum.c:9:14: note: (so you should pass ‘double’ not ‘float’ to ‘va_arg’)
mysum.c:9:14: note: if this code is reached, the program will abort
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

在如下main函数的调用下验证可以获得正确的结果：

```c
int main(int argc, char** argv) {
    float a=1.0, b=2.2, c=9.0;
    printf("%f,%f\n", mysum(2,a,b), mysum(3,a,b,c));
    return 0;
}
```

###思考题

由反汇编代码可得，调用顺序是：系统 ->_start -> __libc_start_main --> main。故main函数也需要保存现场。

为main函数分配的24字节个人猜测是为了内存对齐。

参数的传递顺序在上文已经说过，是从后往前的。

###一些想法

其实，这个实验如果用stdarg来做，真的是再简单不过的了，但不使用stdarg则需要对调用协议做比较深刻的认识。可是这样的程序代码只能在 IA32 架构上运行，因为其他很多的平台均为寄存器传参数，根本无法通过指针来解决问题。所以，这个实验对平台的限定过强。
