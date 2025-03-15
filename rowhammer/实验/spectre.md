## spectre v1 原理

Spectre V1是一种针对处理器硬件分支预测特性的攻击，主要是利用“分支预测会对数据Cache造成影响”这一漏洞，通过侧信道方式绕过处理器安全检查进行间接越界访问，其基本原理如下所示：

![image-20250216165637135](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250216165637135.png)

1. 左图是一般情况下的分支执行情况，程序运行顺序由上至下：先做if判断，然后根据判断结果选择执行if代码段还是else代码段，最后继续运行程序；
2. 右图是分支预测情况下的分支执行情况，程序运行顺序由上至下：先同时执行if代码段和做if判断，然后根据if判断的结果分辨此次分支预测是否正确，若正确则继续运行程序，若错误则执行else代码段（同时丢弃分支预测执行所产生的结果）；
3. 在分支预测执行的过程中，预测错误虽然会丢弃该执行所产生的结果，最终不会对程序可见的寄存器和存储器产生任何影响，但是不会对数据Cache进行回滚冲刷，于是可能对数据Cache产生影响，这时如果进行越界访问且将其相关信息放入数据Cache中，后续就能够通过对数据Cache做侧信道攻击从而逆推出越界访问的内容。

在实际的攻击中，经常要求受害者程序拥有如下所示的特定格式代码段：

![image-20250216165709527](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250216165709527.png)

在该代码段中，变量x为攻击者程序可以控制的恶意输入，变量array1_size指示数组array1的大小，数组array2是一个攻击者也可以访问的大数组。当变量array1_size的Cache状态为Miss时，处理器并不会一直等待array1_size从存储中被取回，而是会触发分支预测机制继续往后执行。于是，攻击者可以通过训练处理器的分支预测器，使得当x不满足if判断条件时也执行if内的语句，由此产生array1的越界访问，该越界访问的值又作为地址的一部分被Cache状态所捕获。此后，攻击者通过访问array2判断其Cache状态，就能够反推出越界访问的值。

## spectre v1限制

**攻击者必须能控制关键变量 X 才能进行攻击。**

而事实上，如果关键变量 X 是来自不可信的来源（例如攻击者的输入），防御措施（如污点追踪或代码序列化）会识别并阻止这种不安全的投机执行。而如果 x 是可信的，即它是由内核生成或控制的，攻击者无法操纵它，攻击自然无法实施。



## 利用rowhammer解除spectre v1 限制

既然由内核生成或控制的 X 是可信的，我们可以间接写入可信的内核变量，即使我们不是特权用户，也可以执行前面所说的攻击。

而rowhammer正好可以做到这一点，使得在不直接控制 X 的情况下，也能利用Row Hammer来执行Spectre攻击，即SpecHammer

<img src="C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250222182422963.png" alt="image-20250222182422963" style="zoom: 80%;" /><img src="C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250222182451062.png" alt="image-20250222182451062" style="zoom: 80%;" />

以上double gadgets，其存在一个限制：由于Row Hammer攻击的特性，攻击每次只能翻转一位，因此很难任意指向受害者内存中的数据。无法像spectre那样灵活地任意瞄准内存中想要泄露的任意值



## Triple Gadgets

由于double gadgets的限制，作者提出了triple Gadgets

![image-20250222183019277](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250222183019277.png)

好处是，不需要翻转位来直接将 X 指向需要泄露的secret。我们只需要翻转x的一个位，使其指向包含我们所控制数据的内存区域。在这个内存区域中，我们可以修改任意想要的值，从而精确地定位想要泄漏的地方。

## 内存模板化（已完成）

![image-20250301161509257](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250301161509257.png)



## 用户空间memory message

1. **Fodder分配**：预先分配一些内存页（称为“fodder”），以便后续释放时能够控制目标页面的位置。
2. **解除页面映射**：释放目标页面，并将其放入PCP列表中。
3. **受害者分配**：通过创建新的进程或线程，使得受害者的栈分配在目标页面上。

```c

// 分配"Fodder"页面
void allocate_fodder(size_t fodder_pages) {
    void **fodder = malloc(fodder_pages * sizeof(void*));
    for (int i = 0; i < fodder_pages; i++) {
        // 1.分配页面并锁定到内存（防止被换出）
		// 2.初始化以触发物理分配
    }
}

// 解除目标页面映射，并释放Fodder
void free_target_and_fodder(void *target_page, void **fodder, size_t fodder_pages) {
    munmap(target_page, PAGE_SIZE); // 释放目标页面到PCP列表
    for (int i = 0; i < fodder_pages; i++) {
        munmap(fodder[i], PAGE_SIZE); // 释放Fodder页面
    }
}

// 3. 触发受害者线程分配栈到目标页面
void *victim_thread(void *arg) {
        // 受害者代码（包含Spectre Gadget）
}

// 主流程
void user_space_massaging() {
    size_t fodder_pages = 100; // 根据配置调整
    target_page // 通过Rowhammer定位的目标物理页
    
    allocate_fodder(fodder_pages);
    free_target_and_fodder(target_page, fodder, fodder_pages);
    
    // 创建受害者线程
    pthread_t tid;
    pthread_create(&tid, NULL, victim_thread, NULL);
    pthread_join(tid, NULL);
}

```

## 内核空间memory message

1. **耗尽内核页面**：通过大量分配内存，迫使内核使用用户页面（MOVABLE类型）。
2. **释放目标页面**：确保目标页面被释放并加入合适的空闲列表。
3. **内核堆栈喷洒**：创建大量线程，增加目标页面被用于内核栈的概率

```c
#include <sys/syscall.h>
#include <linux/userfaultfd.h>
#include <sys/ioctl.h>

// 耗尽内核UNMOVABLE页面（通过大量PTE分配）
void drain_unmovable_pages() {
    int fd = open("/proc/buddyinfo", O_RDONLY);
    // 监控buddyinfo文件，持续分配内存直至UNMOVABLE页耗尽
    while (/* 检测UNMOVABLE剩余量 */) {
            // 填充PTE以消耗内核内存
        }
    }
}

// 利用userfaultfd注入内核可控数据
void inject_kernel_data() {
    // 配置userfaultfd处理函数
    // 分配部分未提交的页面
    // 触发缺页
    // 注册userfaultfd处理缺页
    // 通过syscall（如sendmsg）将数据注入内核栈
}

// 内核栈喷洒
void kernel_stack_spray() {
    // 创建大量线程以分配内核栈
}

// 主流程
void kernel_space_massaging() {
    drain_unmovable_pages();
    inject_kernel_data();
    kernel_stack_spray();
    // 检测目标页面是否成功分配至内核栈
}
```

