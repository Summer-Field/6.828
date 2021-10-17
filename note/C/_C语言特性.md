## static关键字作用

总结：

- 用static声明局部变量，使其变为静态存储方式(静态数据区)，作用域不变，普通的局部变量会被压入栈中，随着函数的结束而结束，但是static修饰的局部变量放在静态存储区，它会随着程序的结束而结束
- 用static声明外部变量，其本身就是静态变量，这只会改变其连接方式，使其只在本文件内部有效，而其他文件不可连接或引用该变量。

## 内联函数

一、什么是内联函数

       在C语言中，如果一些函数被频繁调用，不断地有函数入栈，即函数栈，会造成栈空间或栈内存的大量消耗。
    
       为了解决这个问题，特别的引入了inline修饰符，表示为内联函数。
    
       栈空间就是指放置程式的局部数据也就是函数内数据的内存空间，在系统下，栈空间是有限的，假如频繁大量的使用就会造成因栈空间不足所造成的程式出错的问题，函数的死循环递归调用的最终结果就是导致栈内存空间枯竭。
         上面的例子就是标准的内联函数的用法，使用inline修饰带来的好处我们表面看不出来，其实在内部的工作就是在每个for循环的内部任何调用dbtest(i)的地方都换成了(i%2>0)?"奇":"偶"这样就避免了频繁调用函数对栈内存重复开辟所带来的消耗。
    
     其实这种有点类似咱们前面学习的动态库和静态库的问题，使 dbtest 函数中的代码直接被放到main 函数中，执行for 循环时，会不断调用这段代码，而不是不断地开辟一个函数栈。



二、内联函数的编程风格

1、关键字inline 必须与函数定义体放在一起才能使函数成为内联，仅将inline 放在函数声明前面不起任何作用。

如下风格的函数Foo 不能成为内联函数：

```
inline void Foo(int x, int y); // inline 仅与函数声明放在一起
void Foo(int x, int y)
{
}
```

而如下风格的函数Foo 则成为内联函数：

```
void Foo(int x, int y);
inline void Foo(int x, int y) // inline 与函数定义体放在一起
{
}
```

​       所以说，inline 是一种 “用于实现的关键字” ，而不是一种“用于声明的关键字”。一般地，用户可以阅读函数的声明，但是看不到函数的定义。尽管在大多数教科书中内联函数的声明、定义体前面都加了inline 关键字，但我认为inline 不应该出现在函数的声明中。这个细节虽然不会影响函数的功能，但是体现了高质量C++/C 程序设计风格的一个基本原则：声明与定义不可混为一谈，用户没有必要、也不应该知道函数是否需要内联。

2、inline的使用是有所限制的

      inline只适合涵数体内代码简单的函数数使用，不能包含复杂的结构控制语句例如while、switch，并且内联函数本身不能是直接递归函数(自己内部还调用自己的函数)。


三、慎用内联

       内联能提高函数的执行效率，为什么不把所有的函数都定义成内联函数？如果所有的函数都是内联函数，还用得着“内联”这个关键字吗？
    
       内联是以代码膨胀（复制）为代价，仅仅省去了函数调用的开销，从而提高函数的执行效率。如果执行函数体内代码的时间，相比于函数调用的开销较大，那么效率的收

获会很少。另一方面，每一处内联函数的调用都要复制代码，将使程序的总代码量增大，消耗更多的内存空间。

以下情况不宜使用内联：

（1）如果函数体内的代码比较长，使用内联将导致内存消耗代价较高。

（2）如果函数体内出现循环，那么执行函数体内代码的时间要比函数调用的开销大。

一个好的编译器将会根据函数的定义体，自动地取消不值得的内联（这进一步说明了inline 不应该出现在函数的声明中）。


总结：

       因此,将内联函数放在头文件里实现是合适的,省却你为每个文件实现一次的麻烦.而所以声明跟定义要一致,其实是指,如果在每个文件里都实现一次该内联函数的话,那么,最好保证每个定义都是一样的,否则,将会引起未定义的行为,即是说,如果不是每个文件里的定义都一样,那么,编译器展开的是哪一个,那要看具体的编译器而定.所以,最好将内联函数定义放在头文件中. 



## 指针函数和函数指针

### 概述

指针函数和函数指针是C语言里两个比较绕的概念。但是不仅面试题爱考，实际应用中也比较广泛。很多人因为搞不清这两个概念，干脆就避而远之，我刚接触C语言的时候对这两个概念也比较模糊，特别是当指针函数、函数指针、函数指针变量、函数指针数组放在一块的时候，能把强迫症的人活活逼疯。
其实如果理解了这些概念的本质，是不需要死记硬背的，理解起来也比较容易。

### 指针函数

指针函数： 顾名思义，它的本质是一个函数，不过它的返回值是一个指针。其声明的形式如下所示：

`ret *func(args, ...);`
其中，func是一个函数，args是形参列表，ret *作为一个整体，是 func函数的返回值，是一个指针的形式。
下面举一个具体的实例来做说明：

```c
文件：pointer_func.c

# include <stdio.h>

# include <stdlib.h>

int * func_sum(int n)
{
    if (n < 0)
    {
        printf("error:n must be > 0\n");
        exit(-1);
    }
    static int sum = 0;
    int *p = &sum;
    for (int i = 0; i < n; i++)
    {
        sum += i;
    }
    return p;
}

int main(void)
{
    int num = 0;
    printf("please input one number:");
    scanf("%d", &num);
    int *p = func_sum(num); 
    printf("sum:%d\n", *p);
    return 0;
}

```

上例就是一个指针函数的例子，其中，int * func_sum(int n)就是一个指针函数， 其功能十分简单，是根据传入的参数n，来计算从0到n的所有自然数的和，其结果通过指针的形式返回给调用方。
以上代码的运行结果如下所示：



如果上述代码使用普通的局部变量来实现，也是可以的，如下所示：

```c
文件：pointer_func2.c

# include <stdio.h>

# include <stdlib.h>

int func_sum2(int n)
{   
    if (n < 0)
    {   
        printf("error:n must be > 0\n");
        exit(-1);
    }
    int sum = 0;
    int i = 0;
    for (i = 0; i < n; i++)
    {   
        sum += i;
    }
    return sum;
}

int main(void)
{
    int num = 0;
    printf("please input one number:");
    scanf("%d", &num);
    int ret = func_sum2(num);
    printf("sum2:%d\n", ret);
    return 0;
}

```

本案例中，func_sum2函数的功能与指针函数所实现的功能完全一样。

不过在使用指针函数时，需要注意一点，相信细心地读者已经发现了，对比func_sum和func_sum2函数，除了返回值不一样之外，还有一个不同的地方在于，在func_sum中，变量sum使用的是**静态局部变量**，而func_sum2函数中，变量sum使用的则是普通的变量。
如果我们把指针函数的sum定义为普通的局部变量，会是什么结果呢？不妨来试验一下：

```c
文件：pointer_func3.c

# include <stdio.h>

# include <stdlib.h>

int * func_sum(int n)
{
    if (n < 0)
    {
        printf("error:n must be > 0\n");
        exit(-1);
    }
    int sum = 0;
    int *p = &sum;
    for (int i = 0; i < n; i++)
    {
        sum += i;
    }
    return p;
}

int main(void)
{
    int num = 0;
    printf("please input one number:");
    scanf("%d", &num);
    int *p = func_sum(num); 
    printf("sum:%d\n", *p);
    return 0;
}
```



可是如果我们把main函数里面稍微改动一下：

```c
int main(void)
{
    int num = 0;
    printf("please input one number:");
    scanf("%d", &num);
    int *p = func_sum(num);
    printf("wait for a while...\n");    //此处加一句打印
    printf("sum:%d\n", *p);
    return 0;
}
```


我们在输出sum之前打印一句话，这时看到得到的结果完全不是我们预先想象的样子，得到的并不是我们想要的答案。

为什么会出现上面的结果呢？`sum = 0`
其实原因在于，一般的**局部变量是存放于栈区**的，当函数结束，栈区的变量就会释放掉，如果我们在函数内部定义一个变量，在使用一个指针去指向这个变量，当函数调用结束时，这个变量的空间就已经被释放，这时就算返回了该地址的指针，也不一定会得到正确的值。上面的示例中，**在返回该指针后，立即访问，的确是得到了正确的结果，但这只是十分巧合的情况**，如果我们等待一会儿再去访问该地址，很有可能该**地址已经被其他的变量所占用**，这时候得到的就不是我们想要的结果。甚至更严重的是，如果因此访问到了不可访问的内容，很有可能造成段错误等程序崩溃的情况。
因此，在使用指针函数的时候，一定要避免出现返回局部变量指针的情况。
那么为什么用了static就可以避免这个问题呢？
原因是一旦使用了**static去修饰变量，那么该变量就变成了静态变量**。而**静态变量是存放在数据段**的，它的生命周期存在于整个程序运行期间，只要程序没有结束，该变量就会一直存在，所以该指针就能一直访问到该变量。
因此，还有一种解决方案是使用全局变量，因为全局变量也是放在数据段的，但是并不推荐使用全局变量。

### 函数指针

与指针函数不同，函数指针 的本质是一个指针，该指针的地址指向了一个函数，所以它是指向函数的指针。
我们知道，函数的定义是存在于代码段，因此，每个函数在代码段中，也有着自己的**入口地址**，函数指针就是指向代码段中**函数入口地址的指针**。
其声明形式如下所示：

`ret (*p)(args, ...);`

其中，ret为返回值，*p作为一个整体，代表的是指向该函数的指针，args为形参列表。其中**p被称为函数指针变量** 。

#### 关于函数指针的初始化

与数组类似，在数组中，数组名即代表着该数组的首地址，函数也是一样，函数名即是该数组的入口地址，因此，函数名就是该函数的函数指针。
因此，我们可以采用如下的初始化方式：

`函数指针变量 =  函数名;`

下面还是以一个简单的例子来具体说明一下函数指针的应用：

```c
文件：func_pointer.c

#include <stdio.h>

int max(int a, int b)
{
    return a > b ? a : b;
}

int main(void)
{
    int (*p)(int, int); //函数指针的定义
    //int (*p)();       //函数指针的另一种定义方式，不过不建议使用
    //int (*p)(int a, int b);   //也可以使用这种方式定义函数指针
    

    p = max;    //函数指针初始化
    
    int ret = p(10, 15);    //函数指针的调用
    //int ret = (*max)(10,15);
    //int ret = (*p)(10,15);
    //以上两种写法与第一种写法是等价的，不过建议使用第一种方式
    printf("max = %d \n", ret);
    return 0;

}
```

上面这个函数的功能也十分简单，就是求两个数中较大的一个数。值得注意的是通过函数指针调用的方式。
首先代码里提供了3种函数指针定义的方式，这三种方式都是正确的，比较推荐**第一种和第三种定义方式**。然后对函数指针进行初始化，前面已经提到过了，直接将函数名赋值给函数指针变量名即可。
上述代码运行的结果如下：

调用的时候，既可以直接使用**函数指针调用，也可以通过函数指针所指向的值去调用**。(*p)所代表的就是函数指针所指向的值，也就是函数本身，这样调用自然不会有问题。有兴趣的同学可以去试一试。

#### 为什么要使用函数指针？

那么，有不少人就觉得，本来很简单的函数调用，搞那么复杂干什么？其实在这样比较简单的代码实现中不容易看出来，当项目比较大，代码变得复杂了以后，函数指针就体现出了其优越性。
举个例子，如果我们要实现数组的排序，我们知道，常用的数组排序方法有很多种，比如快排，插入排序，冒泡排序，选择排序等，如果不管内部实现，你会发现，除了**函数名不一样之外，返回值，包括函数入参都是相同的**，这时候如果要调用不同的排序方法，就可以使用指针函数来实现，我们**只需要修改函数指针初始化的地方，而不需要去修改每个调用的地方（特别是当调用特别频繁的时候）。**

#### 回调函数

函数指针的一个非常典型的应用就是**回调函数。**
什么是回调函数？
**回调函数就是一个通过指针函数调用的函数。其将函数指针作为一个参数，传递给另一个函数。**
回调函数并不是由实现方直接调用，而是在**特定的事件或条件发生时由另外一方来调用的。**
同样我们来看一个回调函数的例子：

```C
文件：callback.c

#include<stdio.h>
#include<stdlib.h>

//函数功能：实现累加求和
int func_sum(int n)
{
        int sum = 0;
        if (n < 0)
        {
                printf("n must be > 0\n");
                exit(-1);
        }
        for (int i = 0; i < n; i++)
        {
                sum += i;
        }
        return sum;
}

//这个函数是回调函数，其中第二个参数为一个函数指针，通过该函数指针来调用求和函数，并把结果返回给主调函数
int callback(int n, int (*p)(int))
{
        return p(n);
}

int main(void)
{
        int n = 0;
        printf("please input number:");
        scanf("%d", &n);
        printf("the sum from 0 to %d is %d\n", n, callback(n, func_sum));       //此处直接调用回调函数，而不是直接调用func_sum函数
        return 0;
}
```

上面这个简单的demo就是一个比较典型的回调函数的例子。在这个程序中，回调函数**callback无需关心func_sum是怎么实现的，只需要去调用**即可。
这样的好处就是，如果以后对**求和函数有优化，比如新写了个func_sum2函数**的实现，我们只需要在调用回调函数的地方将函数指针指向func_sum2即可，而无需去修改callback函数内部。
以上代码的输出结果如下：

回调函数广泛用于开发场景中，比如信号函数、线程函数等，都使用到了回调函数的知识。

## register：

这个关键字请求编译器尽可能的**将变量存在CPU内部寄存器**中，而不是通过内存寻址访问，以提高效率。注意是尽可能，不是绝对。

因为，如果定义了很多register变量，可能会超过CPU的寄存器个数，超过容量。所以只是可能。

### 但是使用register修饰符有几点限制。

　　1.register变量必须是能被CPU所接受的类型。这通常意味着register变量必须是一个单个的值，并且长度应该小于或者等于整型的长度。不过，有些机器的寄存器也能存放浮点数。　　　　

　　2.因为register变量可能不存放在内存中，**所以不能用“&”来获取register变量的地址**。由于寄存器的数量有限，而且某些寄存器只能接受特定类型的数据（如指针和浮点数），因此**真正起作用的register修饰符的数目和类型都依赖于运行程序的机器**，而任何多余的register修饰符都将被编译程序所忽略。在某**些情况下，把变量保存在寄存器中反而会降低程序的运行速度**。因为被占用的寄存器不能再用于其它目的；或者变量被使用的次数不够多，不足以装入和存储变量所带来的额外开销。

　　3.早期的**C编译程序不会把变量保存在寄存器中，除非你命令它这样做**，这时register修饰符是C语言的一种很有价值的补充。然而，随着编译程序设计技术的进步，在决定那些变量应该被存到寄存器中时，现在的**C编译环境能比程序员做出更好的决定**。实际上，许多编译程序都会忽略register修饰符，因为尽管它完全合法，但它仅仅是暗示而不是命令。

register关键字在c语言和c++中的差异

> ### 在c++中：
>
> (1)register 关键字无法在全局中定义变量，否则会被提示为不正确的存储类。
>
> (2)register 关键字在局部作用域中声明时，可以用 & 操作符取地址，一旦使用了取地址操作符，被定义的变量会强制存放在内存中。
>
> 在c中:
>
> (1)register 关键字可以在全局中定义变量，当对其变量使用 & 操作符时，只是警告“有坏的存储类”。
>
> (2)register 关键字可以在局部作用域中声明，但这样就无法对其使用 & 操作符。否则编译不通过。
>
> 建议不要用register关键字定义全局变量，因为全局变量的生命周期是从执行程序开始，一直到程序结束才会终止，而register变量可能会存放在cpu的寄存器中，如果在程序的整个生命周期内都占用着寄存器的话，这是个相当不好的举措。
>
> C和C++处理register关键字的一处差异
> 2010年12月29日 星期三 13:30
> 　　C++并不是完全兼容C语言的，今天我在编码时又无意中发现了一处不同：
> 　　用register关键字修饰的变量，在C语言中是不可以用&操作符取地址的，这是我已有的经验。因为编译器如果接受了程序员的建议把变量存入寄存器，它是不存在虚拟地址的。但在C++中，用register修饰的变量可以用&操作符取地址，这是我在一段代码中发现的。如果程序中显式取了register变量的地址，编译器一定会将这个变量定义在内存中，而不会定义为寄存器变量。
> 　　我在C99（ISO/IEC 9899:1999）和ISO C++（ISO/IEC 14882:2003）标准中得到了确认，C和C++标准对register遇到&的处理确实有不同的明确定义。但为什么要这样定义？我只能从标准的字里行间猜测。K&R C1中如何描述register我尚未查证，K&R C2（ANSI C）中说明了“register variables are to be placed in machine registers … but compilers are free to ignore the advice ”。但在C99和ISO C++中，措辞分别变成：“suggests that access to the object be as fast as possible”、“a hint to the implementation that the object so declared will be heavily used”，不再特别提及“machine registers”。可见历史上register关键字在强调尽可能地把变量保存到寄存器，而现在的register关键字不再强调具体手段，只是建议编译器通过各种可行的方式优化该变量的访问（不过很多编译器会忽略这一关键字，而采用自身的优化策略）。C99可能是为了保持对K&R C的兼容而不允许取地址操作；而C++也许是因为没有历史包袱才放宽了这个限制吧。猜测而已，希望知道内幕的朋友告诉我更精确的答案。

## memmov、memcpy

首先来看函数原型：



```cpp
void *memcpy(void *restrict s1, const void *restrict s2, size_t n);
void *memmove(void *s1, const void *s2, size_t n);
```

这两个函数都是将s2指向位置的n字节数据拷贝到s1指向的位置，区别就在于关键字restrict, memcpy假定两块内存区域没有数据重叠，而memmove没有这个前提条件。如果复制的两个区域存在重叠时使用memcpy，其结果是不可预知的，有可能成功也有可能失败的，所以如果使用了memcpy,**程序员自身必须确保两块内存没有重叠部分**。

### memcpy的实现##

一般来说，memcpy的实现非常简单，只需要顺序的循环，把字节一个一个从src拷贝到dest就行：

```cpp
#include <stddef.h> /* size_t */
void *memcpy(void *dest, const void *src, size_t n)
{
    char *dp = dest;
    const char *sp = src;
    while (n--)
        *dp++ = *sp++;
    return dest;
}
```

### memmove的实现##

memmove会对拷贝的数据作检查，确保内存没有覆盖，如果发现会覆盖数据，简单的实现是调转开始拷贝的位置，从尾部开始拷贝:

```cpp
#include <stddef.h> /* for size_t */
void *memmove(void *dest, const void *src, size_t n)
{
    unsigned char *pd = dest;
    const unsigned char *ps = src;
    if (__np_anyptrlt(ps, pd))
        for (pd += n, ps += n; n--;)
            *--pd = *--ps;
    else
        while(n--)
            *pd++ = *ps++;
    return dest;
}
```

这里`__np_anyptrlt`是一个简单的宏，用于结合拷贝的长度检测dest与src的位置，如果dest和src指向同样的对象，且src比dest地址小，就需要从尾部开始拷贝。否则就和memcpy处理相同。
 但是实际在C99实现中，是将内容拷贝到临时空间，再拷贝到目标地址中：

```cpp
#include <stddef.h> /* for size_t */
#include <stdlib.h> /* for memcpy */

void *memmove(void *dest, const void *src, size_t n)
{
    unsigned char tmp[n];
    memcpy(tmp,src,n);
    memcpy(dest,tmp,n);
    return dest;
}
```

由此可见memcpy的速度比memmove快一点，如果使用者可以确定内存不会重叠，则可以选用memcpy，否则memmove更安全一些。另外一个提示是第三个参数是拷贝的长度，如果你是拷贝10个double类型的数值，要写成sizeof(double)*10,而不仅仅是10。

## 可变参函数

 在C/C++中，我们经常会用到可变参数的函数（比如printf/snprintf等），本篇笔记旨在讲解编译器借助va_start/va_arg/va_end这簇宏来实现可变参数函数的原理，并在文末给出简单的实例。
        备注：本文的分析适用于Linux/Windows，其它操作系统平台的可变参数函数的实现原理大体相似。
1. 基础知识
        如果想要真正理解可变参数函数背后的运行机制，建议先理解两部分基础内容：
         1）函数调用栈
         2）函数调用约定
        关于这两个基础知识点，我之前的笔记有详细介绍，感兴趣的童鞋可以移步这里：栈与函数调用惯例—上篇 和栈与函数调用惯例—下篇

2. 三个宏：va_start/va_arg/va_end
        由man va_start可知，这簇宏定义在stdarg.h中，在我的测试机器上，该头文件路径为：/usr/lib/gcc/x86_64-redhat-linux/3.4.5/include/stdarg.h，在gcc源码中，其路径为：gcc/include/stdarg.h。
        在stdarg.h中，宏定义的相关代码如下：

    #define va_start(v,l)  __builtin_va_start(v,l)
    #define va_end(v)      __builtin_va_end(v)
    #define va_arg(v,l)    __builtin_va_arg(v,l)
    #if !defined(__STRICT_ANSI__) || __STDC_VERSION__ + 0 >= 199900L
    #define va_copy(d,s)    __builtin_va_copy(d,s)
    #endif
    #define __va_copy(d,s)  __builtin_va_copy(d,s)
        其中，前3行就是我们所关心的va_start & var_arg & var_end的定义（至于va_copy，man中有所提及，但通常不会用到，想了解的同学可man查看之）。可见，gcc将它们定义为一组builtin函数。
        关于这组builtin函数的实现代码，我曾试图在gcc源码中沿着调用路径往下探索，无奈gcc为实现这组builtin函数引入了很多自定义的数据结构和宏，对非编译器研究者的我来说，实在有点晦涩，最终探索过程无疾而终。在这里，我列出目前跟踪到的调用路径，以便有兴趣的童鞋能继续探索下去或指出我的不足，先在此谢过。
        __builtin_va_start()函数的调用路径：
    // file: gcc/builtins.c
    /* The "standard" implementation of va_start: just assign `nextarg' to the variable.  */
    void std_expand_builtin_va_start (tree valist, rtx nextarg)                        
    {                                                                             
    rtx va_r = expand_expr (valist, NULL_RTX, VOIDmode, EXPAND_WRITE);
    convert_move (va_r, nextarg, 0);  // definition is in gcc/expr.c
    }
    // 上述代码中调用了expand_expr()来展开表达式，我猜测该函数调用完后，va_list指向了可变参数list前的最后一个已知类型参数
    //  file: gcc/expr.h
    /* Generate code for computing expression EXP.
    An rtx for the computed value is returned.  The value is never null.
    In the case of a void EXP, const0_rtx is returned.  
    */
    static inline rtx expand_expr (tree exp, rtx target, enum machine_mode mode,enum expand_modifier modifier)
    {
   return expand_expr_real (exp, target, mode, modifier, NULL);
    }
3. Windows系统VS内置编译器对va_start/va_arg/va_end的实现
        如前所述，我 没能 在gcc源码中 找出va_startva_arg/va_end这3个宏的实现代码（⊙﹏⊙b汗），所幸的是，Windows平台VS2008集成的编译器中，对这三个函数有很明确的实现代码，摘出如下。
    /* file path: Microsoft Visual Studio 9.0\VC\include\stdarg.h */
    #include <vadefs.h>

#define va_start _crt_va_start
#define va_arg _crt_va_arg
#define va_end _crt_va_end
        可见，Windows系统下，仍然将va_start/va_arg/va_end定义为一组宏。他们对应的实现在vadefs.h中：
/* file path: Microsoft Visual Studio 9.0\VC\include\vadefs.h */
#ifdef  __cplusplus
#define _ADDRESSOF(v)   ( &reinterpret_cast<const char &>(v) )
#else
#define _ADDRESSOF(v)   ( &(v) )
#endif

#define _INTSIZEOF(n)   ( (sizeof(n) + sizeof(int) - 1) & ~(sizeof(int) - 1) )

#define _crt_va_start(ap,v)  ( ap = (va_list)_ADDRESSOF(v) + _INTSIZEOF(v) )
#define _crt_va_arg(ap,t)    ( *(t *)((ap += _INTSIZEOF(t)) - _INTSIZEOF(t)) )
#define _crt_va_end(ap)      ( ap = (va_list)0 )
        备注：在VS2008提供的vadefs.h文件中，定义了若干组宏以支持不同的操作系统平台，上面摘出的代码片段是针对IA x86_32的实现。
        下面对上面的代码做个解释：
          a. 宏_ADDRESSOF(v)作用：取参数v的地址。
          b. 宏_INTSIZEOF(n)作用：返回参数n的size并保证4字节对齐（32-bits平台）。这个宏应用了一个 小技巧来实现字节对齐： ~(sizeof(int) - 1)的值对应的2进制值的低k位一定是0，其中sizeof(int) = 2^k，因此，在IA x86_32下，k=2。理解了这一点，那么(sizeof(n) + sizeof(int) - 1) & ~(sizeof(int) - 1)的作用就很直观了，它保证了sizeof(n)的值按sizeof(int)的值做对齐，例如在32-bits平台下，就是按4字节对齐；在64-bits平台下，按8字节对齐。至于为什么要保证对齐，与编译器的底层实现有关，这里不再展开。
          c. _crt_va_start(ap,v)作用：通过v的内存地址来计算ap的起始地址，其中，v是可变参数函数的参数中， 最后一个类型已知的参数，执行的结果是ap指向可变参数列表的第1个参数。以int snprintf(char *str, size_t size, const char *format, ...)为例，其函数参数列表中最后一个已知类型的参数是const char *format，因此，参数format对应的就是_crt_va_start(ap, v)中的v, 而ap则指向传入的第1个可变参数。
        特别需要理解的是：为什么ap = address(v) + sizeof(v)，这与 函数栈从高地址向低地址的增长方向 及 函数调用时参数从右向左的压栈顺序有关，这里默认大家已经搞清楚了这些基础知识，不再展开详述。
          d. _crt_va_arg(ap,t)作用：更新指针ap后，取类型为t的变量的值并返回该值。
          e. _crt_va_end(ap)作用：指针ap置0，防止野指针。
        概括来说，可变参数函数的实现原理是：
         1）根据函数参数列表中最后一个已知类型的参数地址，得到可变参数列表的第一个可变参数
         2）根据程序员指定的每个可变参数的类型，通过地址及参数类型的size获取该参数值
         3）遍历，直到访问完所有的可变参数
        从上面的实现过程可以注意到， 可变参数的函数实现严重依赖于函数栈及函数调用约定（主要是参数压栈顺序），同时，依赖于程序员指定的可变参数类型 。因此，若指定的参数类型与实际提供的参数类型不符时，程序出core简直就是一定的。
4. 程序实例
        经过上面对可变参数函数实现机制的分析，很容易实现一个带可变参数的函数。程序实例如下：

```c
#include <stdio.h>
#include <stdarg.h>

void foo(char *fmt, ...) 
{
    va_list ap;
    int d;
    char c, *p, *s;

    va_start(ap, fmt);
    while (*fmt) 
    {
        if('%' == *fmt) {
            switch(*(++fmt)) {
                case 's': /* string */
                    s = va_arg(ap, char *);
                    printf("%s", s);
                    break;
                case 'd': /* int */
                    d = va_arg(ap, int);
                    printf("%d", d);
                    break;
                case 'c': /* char */
                    /* need a cast here since va_arg only takes fully promoted types */
                    c = (char) va_arg(ap, int);
                    printf("%c", c);
                    break;
                default:
                    c = *fmt;
                    printf("%c", c);
            }  // end of switch
        }  
        else {
            c = *fmt;
            printf("%c", c);
        }
        ++fmt;
    }
    va_end(ap);

}

int main(int argc, char * argv[])
{
    foo("sdccds%%, string=%s, int=%d, char=%c\n", "hello world", 211, 'k');
    return 0;
}
```

## __attribute__

一、介绍

GNU C 的一大特色就是__attribute__ 机制。__attribute__ 可以设置函数属性（Function Attribute ）、变量属性（Variable Attribute ）和类型属性（Type Attribute ）。

__attribute__ 书写特征是：__attribute__ 前后都有两个下划线，并切后面会紧跟一对原括弧，括弧里面是相应的__attribute__ 参数。

__attribute__ 语法格式为：__attribute__ ((attribute-list))

关键字__attribute__ 也可以对**结构体（struct ）或共用体（union ）进行属性设置**。大致有六个参数值可以被设定，即：`aligned, packed, transparent_union, unused, deprecated 和 may_alias` 。

在使用__attribute__ 参数时，你也可以在参数的前后都加上“__” （两个下划线），例如，使用__aligned__而不是aligned ，这样，你就可以在相应的头文件里使用它而不用关心头文件里是否有重名的宏定义。

二、__attribute__ 的参数介绍

**1、aligned** 

指定对象的对齐格式（以字节为单位），如：

```objectivec
struct S {
short b[3];
} __attribute__ ((aligned (8)));
typedef int int32_t __attribute__ ((aligned (8)));
```

该声明将强制编译器确保（尽它所能）变量类 型为struct S 或者int32_t 的变量在分配空间时采用8字节对齐方式。

如上所述，你可以手动指定对齐的格式，同样，你也可以使用默认的对齐方式。如果aligned 后面不紧跟一个指定的数字值，那么编译器将依据你的目标机器情况使用最大最有益的对齐方式。例如：

```objectivec
struct S {
short b[3];
} __attribute__ ((aligned));
```

这里，如果sizeof （short ）的大小为2byte，那么，S 的大小就为6 。取一个2 的次方值，使得该值大于等于6 ，则该值为8 ，所以编译器将设置S 类型的对齐方式为8 字节。

aligned 属性使被设置的对象占用更多的空间，相反的，使用packed 可以减小对象占用的空间。

需要注意的是，attribute 属性的效力与你的连接器也有关，如果你的连接器最大只支持16 字节对齐，那么你此时定义32 字节对齐也是无济于事的。

**2、packed**

​    使用该属性对struct 或者union 类型进行定义，设定其类型的每一个变量的内存约束。就是告诉编译器取消结构在编译过程中的优化对齐（使用1字节对齐）,按照实际占用字节数进行对齐，是GCC特有的语法。这个功能是跟操作系统没关系，跟编译器有关，gcc编译器不是紧凑模式的，我在windows下，用vc的编译器也不是紧凑的，用tc的编译器就是紧凑的。

​    下面的例子中，packed_struct 类型的变量数组中的值将会紧紧的靠在一起，但内部的成员变量s 不会被“pack” ，如果希望内部的成员变量也被packed 的话，unpacked-struct 也需要使用packed 进行相应的约束。

```objectivec
struct unpacked_struct
{
      char c;
      int i;
};

struct packed_struct
{
     char c;
     int  i;
     struct unpacked_struct s;
}__attribute__ ((__packed__));
```

如：

```objectivec
在TC下：struct my{ char ch; int a;}      sizeof(int)=2;sizeof(my)=3;（紧凑模式）
在GCC下：struct my{ char ch; int a;}     sizeof(int)=4;sizeof(my)=8;（非紧凑模式）
在GCC下：struct my{ char ch; int a;}__attrubte__ ((packed))        sizeof(int)=4;sizeof(my)=5
```

下面的例子中使用__attribute__ 属性定义了一些结构体及其变量，并给出了输出结果和对结果的分析。代码为：

```objectivec
struct p



{



    int a;



    char b;



    short c;



 



}__attribute__((aligned(4))) pp;



 



struct m



{



    char a;



    int b;



    short c;



}__attribute__((aligned(4))) mm;



 



struct o



{



    int a;



    char b;



    short c;



}oo;



 



struct x



{



    int a;



    char b;



    struct p px;



    short c;



}__attribute__((aligned(8))) xx;



 



int main()



{           



    printf("sizeof(int)=%d,sizeof(short)=%d.sizeof(char)=%d\n",sizeof(int)



                                                ,sizeof(short),sizeof(char));



    printf("pp=%d,mm=%d \n", sizeof(pp),sizeof(mm));



    printf("oo=%d,xx=%d \n", sizeof(oo),sizeof(xx));



    return 0;



}
```

输出结果：

sizeof(int)=4,sizeof(short)=2.sizeof(char)=1

pp=8,mm=12

oo=8,xx=24

分析：都是字节对齐的原理，可以去看这儿：[字节对齐](https://blog.csdn.net/qlexcel/article/details/79583158)

**3、at**

绝对定位，可以把变量或函数绝对定位到Flash中，或者定位到RAM。

1)、定位到flash中，一般用于固化的信息，如出厂设置的参数，上位机配置的参数，ID卡的ID号，flash标记等等

```objectivec
const u16 gFlashDefValue[512] __attribute__((at(0x0800F000))) = {0x1111,0x1111,0x1111,0x0111,0x0111,0x0111};//定位在flash中,其他flash补充为00



const u16 gflashdata__attribute__((at(0x0800F000))) = 0xFFFF;
```

 

2)、定位到RAM中，一般用于数据量比较大的缓存，如串口的接收缓存，再就是某个位置的特定变量

```objectivec
u8 USART2_RX_BUF[USART2_REC_LEN] __attribute__ ((at(0X20001000)));//接收缓冲,最大USART_REC_LEN个字节,起始地址为0X20001000.
```


注意：

1)、绝对定位不能在函数中定义，局部变量是定义在栈区的，栈区由MDK自动分配、释放，不能定义为绝对地址，只能放在函数外定义。

2)、定义的长度不能超过栈或Flash的大小，否则，造成栈、Flash溢出。

**4、section**

​    提到section，就得说RO RI ZI了，在ARM编译器编译之后，代码被划分为不同的段，RO Section(ReadOnly)中存放代码段和常量，RW Section(ReadWrite)中存放可读写静态变量和全局变量，ZI Section(ZeroInit)是存放在RW段中初始化为0的变量。
​    于是本文的大体意思就清晰了，__attribute__((section("section_name")))，其作用是将作用的函数或数据放入指定名为"section_name"对应的段中。

1)、编译时为变量指定段：

```objectivec
__attribute__((section("name")))



RealView Compilation Tools for µVision Compiler Reference Guide Version 4.0 



 



Home > Compiler-specific Features > Variable attributes > __attribute__((section("name"))) 



 



4.5.6. __attribute__((section("name")))



Normally, the ARM compiler places the objects it generates in sections like data and bss. However, you might require additional data sections or you might want a variable to appear in a special section, for example, to map to special hardware. The section attribute specifies that a variable must be placed in a particular data section. If you use the section attribute, read-only variables are placed in RO data sections, read-write variables are placed in RW data sections unless you use the zero_init attribute. In this case, the variable is placed in a ZI section.



 



Note



This variable attribute is a GNU compiler extension supported by the ARM compiler.



 



Example



/* in RO section */



const int descriptor[3] __attribute__ ((section ("descr"))) = { 1,2,3 };



/* in RW section */



long long rw[10] __attribute__ ((section ("RW")));



/* in ZI section *



long long altstack[10] __attribute__ ((section ("STACK"), zero_init));/
```

2)、编译时为函数指定段

```objectivec
__attribute__((section("name")))



RealView Compilation Tools for µVision Compiler Reference Guide Version 4.0 



 



Home > Compiler-specific Features > Function attributes > __attribute__((section("name"))) 



 



4.3.13. __attribute__((section("name")))



The section function attribute enables you to place code in different sections of the image.



 



Note



This function attribute is a GNU compiler extension that is supported by the ARM compiler.



 



Example



In the following example, Function_Attributes_section_0 is placed into the RO section new_section rather than .text.



 



void Function_Attributes_section_0 (void) __attribute__ ((section ("new_section")));



void Function_Attributes_section_0 (void)



{



    static int aStatic =0;



    aStatic++;



}



In the following example, section function attribute overrides the #pragma arm section setting.



 



#pragma arm section code="foo"



  int f2()



  {



      return 1;



  }                                  // into the 'foo' area



 



  __attribute__ ((section ("bar"))) int f3()



  {



      return 1;



  }                                  // into the 'bar' area



 



  int f4()



  {



      return 1;



  }                                  // into the 'foo' area



#pragma arm section
```

**5、多个属性，组合使用**

```objectivec
u8 FileAddr[100] __attribute__ ((section ("FILE_RAM"), zero_init,aligned(4)));    
```
