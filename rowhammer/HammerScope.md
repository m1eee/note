## 一、Background

研究发现，内存模块功耗越高，翻转一位所需的锤击次数就越多，并提出了一种名为HammerScope的新技术，利用Rowhammer来测量内存模块的功耗。是一种仅使用无特权软件即可测量双列直插式内存模块（DIMM）功耗的侧信道攻击，例如实现精确的指令定时、消除内核地址空间布局随机化（KASLR）的随机性和实施基于功耗分析的“幽灵”（Spectre）攻击。

### 功耗分析

利用了CMOS设备瞬时功耗与这些设备内部状态随时间变化之间的依赖关系。这些状态变化取决于正在处理的指令和数据。因此，能够监测设备功耗的攻击者可以了解其内部状态。要实施功耗分析攻击，攻击者通常会在待测设备（DUT）与其电源之间放置测量探头收集瞬时电流。

#### 1.降低接触要求（Reducing proximity requirement）

用近场电磁探头替代电源探头可以放宽对物理接触的要求，使攻击能够在距离被测设备几毫米的范围内进行。通过测量远场电磁波中的功耗调制，可以进一步放宽要求，将攻击范围扩大到几米

#### 2.消除近距离要求（Reducing proximity requirement）

通过让被测设备自行测量其功耗并将其报告给攻击者，可实现攻击距离的大幅增加。例如，可利用板载模拟-数字转换器（ADC）电路来测量功耗，从而使不受信任的代码能够从TrustZone安全元件中提取密钥



## 二、攻击模型

设存在一个用户级的攻击者，其能够对目标机器反复实施行Rowhammer攻击，并且能够观察攻击是否成功。

不存在额外的CPU或操作系统支持来报告功耗、电池电量、射频辐射或任何其他功耗相关特征。不依赖memory massaging技术。

* 对于DDR3机器，使用HammerTime软件套件中的代码来恢复物理内存与DRAM之间的映射关系，然后应用标准的双面rowhammer来翻转易受攻击的位。
* 对于DDR4机器，使用TRRespass中的代码来恢复映射关系，然后使用9-sided  Rowhammer来翻转位。



## 三、HammerScope

随着 DIMM 功耗的增加，DIMM 中的位更难翻转，需要更多的锤击次数。并且由于DIMM 的功耗与系统活动相关，可以利用行冲刷攻击作为基于功耗的侧信道来监测各种系统事件。

#### 1. 设定激活阈值（Establishing the Activation Threshold）

HammerScope 依赖于利用行翻转攻击来观察 DRAM 的功耗，因此首要任务是确定翻转一位所需的行激活次数的基准。

**1.1 测量方法**：首先在内存模块上进行rowhammer，确定了包含易翻转位的几行。对于每一 行，测量位翻转的概率与行激活次数之间的关系。共测量15 种不同的激活次数，范围在 25 万到 99.5 万之间，对于每个激活次数，都尝试翻转一位100次。

**1.2 测量结果**：激活次数低于某个最小阈值 ACmin时，翻转概率较低

![image-20250126085812652](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250126085812652.png)



#### 2. 观察系统活动（Observing System Activity）

确定ACmin后，便可利用翻转一个位所需的行激活次数来监测系统的内存活动情况。

**观测方法：**运行一个程序，在3 秒内反复向DRAM写入数据，然后让内存空闲10秒。并反复计算翻转一个预先选定的易受行攻击影响的 位所需的行激活次数。激活次数随时间变化如下

![image-20250126090639388](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250126090639388.png)



#### 3. 降低噪音（Reducing Noise）

由于仅使用 ACmin 激活时 Rowhammer的成功概率较低，使得HammerScope无法准确测量DIMM的功耗。因此 尝试用最佳攻击时间来对 HammerScope 观测到的信号进行去噪处理

**提高翻转概率（Increasing Rowhammer Success Probability）**

* **原理：**由于行冲突攻击会增加相邻行的电荷耗尽率，如果一系列Rowhammer攻击在某行的刷新周期之内，并且没有被刷新命令中断，那么攻击成功概率将更高。每行都有一个易受攻击的时间窗口， 在该窗口内启动的行冲突攻击比在窗口外启动的攻击更有可能成功。

* **方法：**对选定的行进行多次行锤击，每次包含固定数量的行激活操作。对于每次尝试 I，记录其开始时间 TI 以及该尝试是否导致位翻转。然后，利用收集到的信息确定刷新周期，得出该行的易受攻击时间窗口。

  ![image-20250126093705511](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250126093705511.png)

* 如上图所示，在正确的刷新周期值下，开始于连续刷新命令之间的特定窗口内的Rowhammer尝试通常成功，而在窗口外的尝试通常失败。将rowhammer攻击与易受攻击窗口同步，将显著降低 HammerScope 测量到的功率信号中的噪声。

  

#### 4.构建HammerScope

* **简单尝试：**在空闲状态下选择一个略高于**ACmin**的激活计数。随后，攻击者执行一个与易受攻击窗口同步的Rowhammer攻击，并在达到所选激活计数后检查目标比特是否翻转。攻击者记录下攻击结果，并进行下一次攻击迭代。这种方法的缺点在于，它仅能产生二元值。一次成功的比特翻转表明系统处于空闲状态，而一次失败的比特翻转则表明DRAM处于活动状态。

* **闭环反馈（Closed-Loop Feedback）**：根据当前攻击迭代翻转目标位的成功 （或失败）情况，自适应地选择下一次攻击迭代的 ACmin 值。根据当前攻击迭代翻转目标位的成功 （或失败）情况，自适应地选择下一次攻击迭代的 ACmin 值。通过这种方法，实质上是追踪 ACmin 值随时间的变化 情况，并将其作为 DIMM 功耗水平的替代指标。这种方法的输出结果并非二进制形式，与前一种方法不 同，而且与猜测并记录的方法相比，其分辨率要精细得多。



#### 6.利用 HammerScope 区分指令(Differentiating Instructions Using HammerScope)

**6.1 利用HammerScope测量指令执行时间**

- 原理：
  - 不同指令的执行时间会影响DRAM的活跃周期（duty cycle）。
  - **长执行时间指令**：降低DRAM的平均活跃周期，导致功耗下降。
  - **短执行时间指令**：提高DRAM的平均活跃周期，增加功耗。
- 测量方法：
  - 在一个核心上执行DRAM激活操作（访问未缓存地址），随后执行目标指令。
  - 在另一个物理核心上，通过HammerScope连续测量DIMM的功耗。

* **结果**：

  * 不同指令的执行时间差异导致功耗变化，可以通过HammerScope清晰区分三种指令类型，如图所示

    ![image-20250126113831541](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250126113831541.png)

  * HammerScope 也可区分**可变时间指令**，如执行不同除数的除法

  ![image-20250126113924524](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250126113924524.png)

  * 使用HammerScope测量**预取命令（Prefetch Commands）**
    - **有效地址**：Prefetch命令快速执行，因为TLB中已存储有效地址的翻译。
    - **无效地址**：Prefetch命令慢速执行，因为需遍历页表来获取地址翻译。
    - 执行时间长的Prefetch命令降低DRAM活跃周期，导致功耗下降。



## 攻击方法

以下内容利用Hammerscope发动端到端攻击

#### 1.区分已映射地址和未映射地址

HammerScope能够测量Prefetch命令的执行时间，而且Prefetch命令的执行时间取决于地址是已映射还是未映射。具体地：当每次迭代中执行的预取命令次数足够多时，针对映射地址快速终止的预取命令与针对未映射地址较慢的预取命令之间的运行时间差异就会通过 DRAM 功耗表现出来。

#### 2.攻击内核地址空间布局随机化（KASLR）

* HammerScope通过Prefetch命令区分映射地址和未映射地址。

* 利用`PREFETCHNTA`指令预取所有可能的内核基地址。未映射地址的Prefetch命令较慢，导致低激活计数。映射到内核的地址Prefetch命令较快，导致高激活计数

* 每次预取地址后，通过内存读操作和Rowhammer测量成功比特翻转的激活阈值，判断地址是否映射到内核。从而定位内核基地址，打破KASLR

  ![image-20250126123700013](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250126123700013.png)

#### 3.HammerScope与Spectre相结合

* 使用HammerScope确定内核的内存布局（如通过破解KASLR）。

* 在已知内核内存布局的前提下，定位存在Spectre v1 gadget的代码位置。

* 利用Spectre v1 gadget推测性地读取敏感值`secret`。

* 通过缓存侧信道泄漏这些敏感信息。



## 局限性及应对措施

#### 1.局限性

**1.1低采样率（ LowSamplingRate）：**HammerScope的采样率仅为15–32 Hz（见表2），远低于低端示波器的采样率。低采样率限制了针对高级攻击（如差分功耗分析，DPA）的能力，特别是在对抗带有侧信道防护的实现时。

**1.2垂直精度低（LowVerticalAccuracy）：**HammerScope的功耗测量精度显著低于专用模数转换器（ADC）。这要求功耗信号的能级差异足够大，才能通过HammerScope区分。

**1.3无法攻击常时间指令：**

HammerScope目前无法从常时间（constant-time）指令中提取信息，例如`aes-ni`或整数乘法操作。

**1.4需要代码执行：**

HammerScope作为一种Rowhammer攻击，依赖于在目标机器上执行非特权代码。而某些设备（如智能卡、硬件安全模块（HSM）和专有微控制器）可能不允许代码执行。

#### 2.防御措施

**2.1 编写恒功耗代码（Constant-Power Code）：**安全敏感代码的功耗与指令或数据无关，从根本上防止秘密信息从功耗泄露。

**2.2 增加噪声（Adding Noise）：**通过引入随机噪声降低信号与噪声比（SNR），使攻击难以提取有效信息。

**2.3 防御Rowhammer攻击（Rowhammer Mitigation）**

