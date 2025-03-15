## 攻击者模型：

1. 与受害者进程共享的数组array2

2. 能获取受害者程序中canary与共享数组array2的相对地址

3. 存在以下代码片段： 

   ```
   if (x < array1_size) {
   	junk &= array1[x];
   	junk &= data->array2[array1[x]];
   }
   ```

4. 能控制x的值（利用rowhammer）

5. 无需特殊权限



## 攻击流程

1. **共享内存建立：**
   受害者与攻击者均通过 `shm_open` 和 `mmap` 映射同一块共享内存，内含结构体 `SharedData`。该结构体包含了大数组 `array2`、控制变量 `trigger_index` 和敏感数据偏移量 `malicious_x`。

```cpp
struct SharedData {
    uint8_t array2[4096 * 256];
    volatile size_t trigger_index;
    size_t malicious_x;
};
```

2. **受害者初始化：**
   受害者程序初始化了一个小数组 `array1` 并设置了其大小，同时计算 `malicious_x`（即 canary 地址与 `array1` 起始地址之间的差值）。在无限循环中，受害者根据共享内存中的 `data->trigger_index` 对 `array1` 和 `array2` 进行访问。
3. **攻击者训练分支预测：**
   攻击者程序利用反复设置 `trigger_index` 的方式训练受害者的分支预测机制，使得在特定时机受害者的执行路径发生偏差，从而触发spectre漏洞。

4. **利用spectre漏洞：**
   由于对 `trigger_index` 缺乏严格的边界检查，当其被外部修改为恶意值时，将产生越界访问，将访问的信息放入Cache中。

5. **cache时序测量：**
   攻击者通过 `_mm_clflush` 清空 `array2` 中的缓存行，并利用时间戳指令（`__rdtscp`）测量每次访问 `array2` 的时延。通过比较命中与未命中时延，确定哪些缓存行被投机加载。

6. **逐字节推断敏感数据：**
   每次通过多次测量后，攻击者可以确定哪一个字节被加载到缓存中，并依此推断出敏感数据。这一过程重复进行，直至重构出完整的canary值。

## victim.cpp



## attack.cpp



## 攻击效果

![image-20250308193700544](C:\Users\MIE\Desktop\主仓库\图片\image-20250308193700544.png)

