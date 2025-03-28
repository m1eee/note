## 一、核心贡献

#### **1. 双代码片段攻击（Double Gadgets）**

- **背景**：传统的 Spectre v1 攻击需要攻击者能够直接控制受害者代码中使用的数组偏移量（offset）。这种要求限制了可以利用的 Spectre gadgets 数量，尤其是在严格受控的环境中。
- 突破
  - SpecHammer 使用 **Rowhammer 位翻转** 修改受害者的偏移量变量，即使攻击者无法直接访问变量，也能通过投机执行泄露敏感数据。
  - 利用 Rowhammer 攻击消除了 Spectre v1 的主要限制，显著扩大了攻击面。例如，在 Linux 5.6 内核中，利用 Rowhammer 位翻转后，gadgets 的数量从约 100 增加到了 20,000。
- 结果
  - **攻击目标**：可以泄露如stack canaries等关键数据。
  - **影响**：这种攻击方法使原本防御严密的代码也能受到Spectre v1攻击。

------

#### 2. **三代码片段攻击（Triple Gadgets）**

- **背景**：传统 Spectre gadgets 通常只能访问有限的内存范围，无法实现任意内存读取。攻击者需要多次精准翻转特定位来扩展攻击范围。
- 突破
  - SpecHammer 引入了“三重嵌套”代码片段（triple gadgets），通过单次 Rowhammer 位翻转实现任意内存读取：
    - 三代码片段的逻辑：代码中包含三层嵌套数组访问，如 `array2[array1[array0[x]]]`。
    - 通过 Rowhammer 改变偏移量，使其指向攻击者控制的数据，从而间接访问任意内存地址。
  - 攻击者仅需翻转一个位即可控制数组偏移，极大简化了攻击复杂度。
- 结果
  - **任意读取能力**：利用内核中的三代码片段，可以读取内核地址空间中的任意数据。
  - 绕过防御
    - **SMAP（Supervisor Mode Access Prevention）**：通过将攻击者控制的数据注入内核堆栈，绕过内核对用户空间数据的访问限制。
    - **KASLR（Kernel Address Space Layout Randomization）**：由于内核中攻击点和攻击者数据之间的相对位置固定，因此地址随机化无法防御此攻击。

------

#### 3. **克服的关键挑战**

SpecHammer 的实现中面临多种技术难点，研究团队提出了创新解决方案：

##### 3.1 **生成足够的 Rowhammer 翻转**

- **问题**：传统方法中，Rowhammer 翻转速率低，限制了攻击可用性。
- 改进
  - 通过解决缓存行刷新的问题（例如，未正确清除缓存导致位翻转检测被掩盖），翻转检测率提高了 **525 倍**（DDR3）和 **16 倍**（DDR4）。
- **影响**：这使得 SpecHammer 攻击的可行性大大提高。

##### 3.2 **内存模板化与栈调整**

- **问题**：需要确保受害者的关键变量分配在 Rowhammer 易受攻击的位置。

- 解决方案

  - 开发了新的栈调整（Stack Massaging）技术：
    - **用户空间栈调整**：通过内存分配和释放控制栈变量的分配地址。
    - **内核空间栈调整**：利用内存压力和分页分配策略，精确控制内核栈的物理地址。
  
- **结果**：攻击者能够有效地将偏移变量强制分配到含有目标位翻转的内存位置。

##### 3.3 **在实际代码中寻找 gadgets**

- **问题**：需要在真实代码中寻找符合攻击条件的代码片段。

- 解决方案：

  - 使用扩展的工具（如 Smatch）扫描 Linux 内核代码。
- **结果**：发现了超过 **20,000 个双代码片段**和 **170 个三代码片段**，极大地扩展了攻击范围。

------

#### 4. **扩展的攻击范围与实验证明**

- 攻击范围的显著扩展
  - Linux 内核中的 gadgets 数量显著增加。
  - 攻击者不再需要直接控制变量输入，消除了 Spectre 的主要限制。
- 概念验证实验                            
  - 演示了双代码片段攻击如何泄露堆栈金丝雀，以及三代码片段攻击如何实现任意内存读取。
  - 在 DDR3 上的泄露速率高达 **24b/s**，在 DDR4 上为 **6b/min**，均实现了 **100% 的准确率**。



## 二、攻击原理

### 1. 双gadget攻击（Double Gadget Attack）

#### 1.1攻击原理

Spectre v1的一个关键限制是攻击者必须控制用作受害者数组索引的变量，可以通过使用 Rowhammer在不直接访问的情况下修改索引变量来解决。

```c++
 1 if(x < array1_size){
 2 	 victim_data = array1[x]
 3   z = array2[victim_data * 512];
 4 }
```

#### 1.2**攻击步骤**

##### **①内存模板化（Memory Templating）**

- Rowhammer 攻击的第一步是定位易受比特翻转影响的物理地址。
- 攻击者通过反复访问内存，寻找可以触发有用比特翻转的内存行（如使索引变量 `x` 指向秘密数据的地址）。
- 找到合适的地址后，利用“内存布局技术”将目标变量 `x` 放置到受控物理地址中，使其易受 Rowhammer 翻转影响。

##### **② 分支预测训练（Branch Predictor Training）**

- 攻击者运行受害者的代码，确保分支预测器学习到合法情况下 `if` 语句的执行路径（即 `x < array1_size` 时进入分支）。
- 这种训练确保 CPU 预测未来代码将按同样路径执行。

##### **③Rowhammer 翻转与错误推测（Hammering and Misspeculation）**

- 攻击者触发 Rowhammer 攻击，诱发 `x` 的比特翻转，使其值增加到超出 `array1` 的边界，指向秘密数据。
- 同时，攻击者清空缓存中的 `x` 值，确保接下来的读取会从内存中使用翻转后的值。
- 当触发受害者代码时，CPU 根据分支预测继续推测执行，使用被翻转的 `x` 值访问 `array1`，加载秘密数据到 `array2` 的索引中，并将秘密数据相关的内存块加载到缓存中。
- 最终，尽管 CPU 检测到推测执行错误并回滚，缓存中的秘密数据相关状态不会被清除。

##### **④ 缓存侧信道恢复数据（Flush+Reload）**

- 攻击者通过 

  Flush+Reload

   技术读取泄露的秘密数据：

  - 攻击者依次访问 `array2` 的每个值并测量访问时间。
  - 如果访问时间较快，说明对应的缓存块已经被加载。
  - 通过这种方式，攻击者可以推断出秘密数据的值。

![image-20250122140746903](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250122140746903.png)



### 2.三重gadget攻击（Triple Gadget Attack）

#### 2.1**攻击原理**

利用三层嵌套数组访问来绕过 Rowhammer 比特翻转限制的攻击技术。与双Gadget攻击不同，该方法仅需要一次比特翻转即可实现对任意内存数据的读取。通过修改三重嵌套数组访问中的内部偏移值，攻击者可以让数组索引指向可控数据，从而间接访问任何内存地址。

```c++
cif (x < array1_size) {
    attacker_offset = array0[x];
    victim_data = array1[attacker_offset];
    y = array2[victim_data * 512];
}
```

1. **三重嵌套访问**：
   - 原始 Spectre v1 假设攻击者控制变量 `x`，用于嵌套访问两个数组（例如 `array2[array1[x]]`）。
   - 三重Gadget攻击扩展了这种机制，支持对三层嵌套数组的访问（例如 `array2[array1[array0[x]]]`）。
2. **利用 Rowhammer 翻转**：
   - 通过触发单个比特翻转，使 `array0[x]` 指向攻击者控制的数据。
   - 攻击者控制的偏移值可进一步指向任意内存地址，攻击者可以修改最内部的数组偏移量x，使array0[x]指向攻击者控制的数据，然后攻击者能够向array2[array1[]]发送任意偏移量，实现对目标地址空间的任意读取。

#### 2.2**攻击步骤**

**①：内存分析与分支预测训练**

- **内存分析**：定位物理内存中易受比特翻转影响的地址，确保找到合适的翻转位置。
- **分支预测训练**：运行受害者代码，训练分支预测器，使其预测 `if` 语句通常会执行（即 `x < array1_size` 为真）。

**②：Rowhammer 翻转与错误推测**

- **比特翻转**：通过 Rowhammer 攻击使变量 `x` 的值超出 `array0` 的边界，指向攻击者控制的数据。
- 错误推测执行
  1. 在缓存中逐出 `array1_size` 的值，确保 CPU 从内存中加载翻转后的值。
  2. 触发受害者代码，在推测执行过程中，使用翻转后的 `array0[x]` 作为偏移值。
  3. CPU 按照 `array1[array0[x]]` 加载数据，攻击者可通过控制 `array0[x]` 决定访问的目标地址。

**③：Flush+Reload**

- 使用缓存侧信道技术（Flush+Reload）检测推测执行中加载的内存数据。
- 攻击者通过计时每个缓存块的访问时间，推断出与秘密数据相关的缓存块位置。



## 三、内存模板化与翻转率改进

内存模板化的核心目的是找到内存中易受 **Rowhammer** 攻击的比特翻转位置，这些“有用”的翻转可用于攻击中改变数组偏移量，从而实现对秘密数据的访问。

1. **目标**：定位能触发比特翻转的物理地址，这些翻转将使目标变量（如数组索引）指向敏感数据。
2. **步骤依赖**：模板化的结果直接影响内存布局调整步骤（memory massaging），因为目标变量需被强制放置在可被翻转的位置。

### **技术实现**

#### **1. 从虚拟地址到DRAM地址的映射**

**1.1难点**：非特权攻击者无法直接看到虚拟地址与 DRAM 行之间的映射，因此需反向推导。

**1.2 技术方法**

- 物理地址到 DRAM 映射：使用 **Pessl 的 DRAMA 技术** 解析虚拟地址到物理地址的映射。
- 虚拟地址到物理地址的映射：对于 DDR3 系统（如 Haswell 处理器），仅需提取物理地址的低 21 位即可，可以通过 **RAMBleed 技术**来获取连续的 2MiB 页，并从中提取这 21 位物理地址位。对 DDR4 内存，使用 **TRRespass 方法**获取映射。

#### **2. 内存行的翻转配置**

**2.1 翻转配置**

- 初始化受害行（victim rows）为全 0 值，并将攻击行（aggressor rows）设置为全 1 值，形成 1-0-1 的攻击模式。反复从缓存中读取和刷新攻击行，确保每次读取直接访问 DRAM，并对相邻行造成扰动效应。完成固定数量的读取后，读取受害行以检查是否发生比特翻转，

**2.2诱导翻转**

- 重复访问并清空缓存，确保每次内存读取直接访问 DRAM，从而引发相邻行的干扰效应。
- 检查受害行中的比特翻转是否可用于攻击（如将数组偏移指向秘密数据）。

### **现有技术的不足与改进**

#### **现有问题**

1. 翻转率低
     - 在 DDR3 内存中，使用已有 Rowhammer 工具的比特翻转率为每小时 2-5 次，远低于攻击所需的效率。
  2. 缓存影响未考虑
     - 现有工具忽略了cache中存储的初始化数据，导致受害行的翻转数据被掩盖，翻转率被严重低估。

#### **解决方案**

**增加翻转率**：修改现有工具，确保在每次检查受害行翻转前，清空受害行的缓存数据。实验表明，翻转率的提高是由于cache刷新后直接读取 DRAM 数据，而非cache数据。

  ### **结果与比较**

  1. **翻转率改进**：
   - 修改后的方法可在 DDR3 DIMMs 上每小时产生数千次翻转，而未修改的工具仅能找到极少翻转。
     - 在 DDR4 DIMMs 上，改进后的工具同样提升了翻转效率，支持对现代内存的攻击。
2. **对比分析**：
     - 现有方法（如 Rowhammer.js 和 TRRespass）在未修改时受到缓存掩盖影响，翻转检测能力有限。
   - 改进后的工具克服了这些限制，显著提升了翻转发现率。



## 四、栈调整（StackMassaging）

在获得一个有用的、易受翻转的地址后，下一步是将受害变量强制放置在该地址上。目标受害变量是一个作为数组偏移量使用的变量。这类变量通常被分配为局部变量，因此位于受害者的栈上。为了翻转这些变量并触发攻击，要将受害者的栈放置在模板化步骤中获得的易受翻转的页面上，此步骤用到StackMassaging技术。主要目标是释放当前由攻击者持有的易受翻转页面，然后强制受害者的分配使用该刚刚释放的页面。

### 1. 用户空间栈调整

**步骤 1：填充分配**
首先，我们进行“填充”分配，以抵消受害者在分配栈之前可能进行的任何分配。目标变量可能并不位于受害者栈的第一页上。因此，我们需要首先计算受害者在分配包含目标变量的栈页面之前将使用多少页面，并分配这些数量的填充页面。

**步骤 2：取消映射页面**
接着，我们释放易受翻转页面，将其放入 PCP 列表中，然后释放填充页面，将它们放在相同列表中并位于易受翻转页面之上。

**步骤 3：受害者分配**
最后，我们启动受害者进程，强制其执行预测的分配，包括目标栈的分配。在目标分配之前发生的任何分配都会从 PCP 列表中移除填充页面，从而强制栈使用目标页面。

### 2. 内核空间栈调整

攻击者需要一种方法来强制内核使用“用户页面”（MOVABLE 页面）而非“内核页面”（UNMOVABLE 页面）。当内存压力较大时，且内核会使用用户页面。因此内核栈调整所需的步骤与用户空间栈调整类似，区别在于攻击者必须施加内存压力，迫使内核使用用户页面。

**耗尽用户页面（Step 2: Draining User Pages）**

- 通过分配和释放内存，使内核的最大可用 **MOVABLE** 内存块缩小到包含目标页面（易受翻转页面）。
- 耗尽高阶块（顺序 ≥ 4），避免超出内存水印限制。

**释放目标页面（Step 3: Freeing Target Page）**

- 释放目标页面及其“伙伴”页面，确保目标页面合并为最大的可用块。
- 通过释放额外页面，填满 PCP 列表，使目标页面被逐出到伙伴分配器的空闲列表。

**分配内核栈（Step 4: Allocating Kernel Stack）**

- 通过内存分配，迫使内核栈使用包含目标页面的块。
- 使用“栈喷洒（kernel stack spray）”技术生成大量线程，通过多次分配耗尽 **UNMOVABLE** 页面，确保目标页面被分配给内核栈。

**检测成功**

- 逐个测试线程，检查是否成功利用目标页面。如果未成功，重复上述过程。





## 补充：伙伴分配器（buddy allocator）

伙伴分配器（Buddy Allocator）是一种**内存管理算法**，主要用于操作系统或内存分配器中，以有效管理内存的分配和释放。它通过将内存划分为大小相等且彼此独立的块（称为“伙伴”）来工作。

### **工作原理：**

- 内存最初被划分为一个大块，其大小通常是2的幂（比如 2^n）。
- 当需要分配内存时，分配器会找到满足需求的最小块：
  - 如果找到的块比需求大，则将块一分为二，直到找到适合的块为止。
  - 这些分裂后的块被称为“伙伴”。
- 当内存释放时，分配器会检查释放块的“伙伴”是否也空闲：
  - 如果是，则将它们合并为更大的块（类似于逆向操作）。
  - 这个过程可以重复，最终可能将内存恢复为最初的一个大块。

当系统内核初始化完毕后，使用页分配器管理物理页，当使用的页分配器是伙伴分配器，伙伴分配器的特点是算法简单且高效。连续的物理页称为页块（page block）。阶（order）是伙伴分配器的一个专业术语，是页的数量单位，2^n 个连续页称为n阶页块。物理内存被分成多个order:0 ~ 10,每个order中连续page的个数是2order,如果一个order中可用的memory size小于期望分配的size，那么更大order的内存块会被对半切分，切分之后的两个小块互为buddies。其中一个子块用于分配，另一个空闲的。这些块在必要时会连续减半，直到达到所需大小的memory 块为止，当一个block被释放之后，会检查它的buddies是否也是空闲的，如果是，那么这对buddies将会被合并。

满足以下条件 的两个n阶页块称为伙伴（buddy）：

1、两个页块是相邻的，即物理地址是连续的；

2、页块的第一页的物理页号必须是2n的整数倍；

3、如果合并成（n+1）阶页块，第一页的物理页号必须是 （2）^n + 1 的整数倍。

伙伴分配器分配和释放物理页的数量单位为阶。分配n阶页块的过程如下：

1、查看是否有空闲的n阶页块，如果有直接分配；否则，继续执行下一步；

2、查看是否存在空闲的（n+1）阶页块，如果有，把（n+1）阶页块分裂为两个n阶页块，一个插入空闲n阶页块链表，另一个分配出去；否则继续执行下一步。

3、查看是否存在空闲的（n+2）阶页块，如果有把（n+2）阶页块分裂为两个（n+1）阶页块，一个插入空闲（n+1）阶页块链表，另一个分裂为两个n阶页块，一个插入空间(n阶页块链表，另一个分配出去；如果没有，继续查看更高阶是否存在空闲页块。

![image-20250123190052379](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250123190052379.png)

上图的buddy分配和回收可以简单叙述成：

- 需要分配内存时如果没有合适的内存块，会对半切分内存块直到分离出合适大小的内存块为止，最后再将其返回。

- 需要释放内存时，会寻找相邻的块，如果其已经释放了，就将这俩合并，再递归这个过程，直到无法再合并为止

![image-20250124104823498](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250124104823498.png)
