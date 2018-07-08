今天在写文件系统时，想在编译时检查一些struct的大小，就发现了这篇blog，讲得是内核里面的BUILD_BUG_ON的实现，这个优雅精妙的实现让我对内核开发者的敬佩之情真是油然而生啊！！

有时候，我们在写C程序的时候需要对struct的大小做一些限制。比如说，struct需要以某些字节大小进行对齐，以符合硬件的支持（也许是某个设备的DMA缓冲区，要使用地址的低位做一些其他的计算），并且这些struct组成了一个数组，每个数组里面的struct都需要以字节8对齐。

有时候我们可以这样写

#if ((sizeof(struct mystruct) % 8 ) != 0)
#error "you screwed up struct mystruct again!"
#endif
但是这样不一定会成功，预处理器不一定会支持sizeof操作符，也不知道C的具体类型。或者，可以把它写到运行时代码里面：
if ((sizeof(struct mystruct) % 8 ) != 0) {
    printf("You screwed up mystruct again!\n");
    exit(1);
}
但是这样会产生额外的代码，消耗不必要的CPU时间，并且这个问题直到运行时才会发现这个BUG。
其实Linux内核里面已经有相关的macro解决这个问题，在include/linux/kernel.h：

#define BUILD_BUG_ON(condition) ((void)sizeof(char[condition? 1:-1])
#define BUILD_BUG_ON(condition) ((void)sizeof(char[1 - 2*!!(condition)]))
使用这个黑科技，我们可以在编译时就对struct大小进行检查
int main()
{
    /* check that you didn't screw up mystruct */
    BUILD_BUG_ON(sizeof(struct mystruct) % 8));
    return 0;
}
如果struct不是8的倍数，这个在编译时会报错；如果它是struct的倍数，这个语句不会产生任何运行时代码。完美符合我们的要求。
但是它是怎么运行的呢？

比如说，

 sizeof(char[1]); /* this gives you the size of a */
                  /* character array of 1. */
 sizeof(char[-11]); /* this doesn't compile */
这两条语句，第一条不会报错，第二条会报错。基于这个原理，我们就可以设计这个struct的sizeof检查了。
PS：下面这些我参考原blog简要解释以下BUILD_BUG_ON的实现，就不一一翻译了

!!(condition)
这个语句强制让condition返回0或者1.
1 - 2*!!(condition)
这个就是条件符合时，大小为0；，编译不报错；否则大小为-1，编译报错。
1;
这个语句不会产生额外的语句，但是会有编译时的警告

(void)1;
这个语句即使使用了 -Wall的编译参数，也不会有警告。
因此，BUILD_BUIG_ON这个宏展开后等效于

(void)sizeof(char[1])
或者
(void)sizeof(char[-1])
