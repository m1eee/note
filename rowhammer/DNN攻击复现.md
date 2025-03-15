**要求**
OS：Ubuntu 20.04
Pytorch：1.12.1+cu113
Torchvision：0.13.1+cu113

------

### 1. 训练 DNN 模型并导出模型参数

#### 1.1 使用 Python 训练 DNN 模型

**数据集：**

- GTSRB

**步骤 1：进入 train_on_torch 文件夹。**

```bash
cd infras
```

**步骤 2：准备训练数据。**
为了构建统一的训练和测试数据读写格式，我们使用 torchvision.datasets 提供的 ImageFolder API 将所有数据集保存为 .png 格式的图片

该功能在 `prepare_datasets.py` 中实现，只需运行：

```bash
python prepare_dataset.py --dataset cifar10
```

执行此步骤后，获得一个适用于 ImageFolder 读取图像数据并转换为 dataloader 的重新整理的文件夹。

接着运行 `train.py` 以获取训练好的 DNN 模型权重参数。

```bash
python train.py --network vgg16 --dataset gtsrb
```

![image-20250301161404136](C:\Users\MIE\Desktop\主仓库\图片\image-20250301161404136.png)


训练好的 DNN 模型参数存储在 `../trained_models` 文件夹中。训练得到的 pytorch 权重参数命名为 `best.task.3.ckpt`，并根据网络和数据集分类存储在 `../trained_models` 中。

**步骤 3：导出训练好的模型参数为 .txt 格式。**
该功能在 `export_model.py` 文件中实现。

```bash
python export_model.py --network vgg16 --dataset gtsrb
```

![](图片/屏幕截图%202025-03-06%20150244.png)

如图，train.py源代码export_network函数在运行时会因为占用内存过大导致卡死，需要进行修改如下：

```python
import tqdm
def export_network(args):
    model_folder = args.dataset + "_" + args.network
    model_abs_path = os.path.join(args.save_root, model_folder)
    model_path = os.path.join(model_abs_path, args.best_name)
    model = torch.load(model_path, map_location=torch.device('cpu'), weights_only=False)
    model = model.state_dict()
    output_path = os.path.join(model_abs_path, args.output_name)
    np.set_printoptions(precision=8, threshold=100)
    total_params = len(model)
    with open(output_path, "a") as out_f:
        # 使用 tqdm 包装迭代器，显示进度条
        for name, tensor in tqdm(model.items(), total=total_params, desc="Exporting", unit="param"):
            shape = tensor.shape
            value = tensor.cpu().numpy()
            out_f.write(f"{name}{shape}\n")
            np.savetxt(out_f, value.flatten().reshape(-1, 1), fmt="%.8f")
            # 每写完一批数据后，清理内存
            del value
```

![屏幕截图 2025-03-06 151138](C:\Users\MIE\Desktop\主仓库\图片\屏幕截图% %2025-03-06 151138.png)
![](图片/屏幕截图%202025-03-06%20151138.png)
执行后，模型参数将以浮点数格式导出，并存储在 `../trained_models/gtsrb_vgg16/best.task.txt` 文件中。同时，脚本会输出模型测试正确率

![image-20250301161818426](C:\Users\MIE\Desktop\主仓库\图片\image-20250301161818426.png)



### 2. 运行 ML 推理

**要求**

- gcc：gcc (Ubuntu 9.4.0-1ubuntu1~20.04.1) 9.4.0
- make：GNU Make 4.2.1
- OpenBLAS：0.3.20

#### 2.1 安装 OpenBLAS

进入 `poc` 文件夹：

```bash
cd poc
```

运行 `make`

```bash
make
```

#### 2.2 运行 MLInfras

在此步骤中，将使用导出的模型参数通过 `poc/infras` 文件夹中实现的 ML 推理基础设施进行推理。

```bash
cd poc/infras
make gtsrb_vgg16
```

之后，导出的模型参数将被加载，并在相同的测试集上进行评估。



### 3. 攻击

**要求**

- cmake：3.16.3
- clang：ubuntu clang version 12.0.0-3ubuntu1~20.04.5

#### 3.1 安装 LLVM

```bash
cd ~
wget https://apt.llvm.org/llvm.sh
chmod +x llvm.sh
sudo ./llvm.sh 12
```

**更新软链接**

```bash
clang --version
which clang
ls -l /usr/bin | grep clang
ln -s /usr/lib/llvm-12/bin/clang /usr/bin/clang
```

对 clang++ 和 opt 也执行相同的更新过程。



#### 3.2 编译 llvm-pass

 LLVM-VIS 算法核心功能以 llvm-pass 的形式实现（路径为 `attack\source\ground_truth\llvm-pass\flipBranches` 和 `attack\source\ground_truth\llvm-pass\flipLauncher`），因此首先需要将该 llvm-pass 编译为 .so 库。

在文件 `flipLauncher\flipLauncher.cpp` 中有两个绝对路径需要替换为绝对路径：

第 20 行：

```
#define BRANCH_INFO_FILE "/home/xxx/SGXBLAS/attack/source/ground_truth/br_info.tmp"
```

第 79 行：

```cpp
std::string command_prefix = "opt -enable-new-pm=0 -load ${HOME}/xxx/SGXBLAS/attack/source/ground_truth/llvm-pass/build/flipBranches/libflipBranches.so -flipbranches -o ";
```

检查完毕后，开始编译 llvm-pass：

```bash
cd attack\source\ground_truth\llvm-pass
mkdir build && cd build 
cmake ..
make 
```



#### 3.3 运行攻击实例

检查编译后的 LLVM 路径，将下列文件中的路径 `${HOME}/xxx/SGXBLAS` 替换为绝对路径（有多处）：

```bash
ground_truth/openblas_makefiles/attack/dirvier/level3/Makefile

ground_truth/openblas_makefiles/attack/interface/Makefile
```

在攻击之前，需要在 `attack\source\ground_truth\Makefile` 的第 34、35、36 行指定攻击实例：（原文件已改好）

```makefile
EXP_CONF = gtsrb_vgg16
EXP_DATASET = gtsrb
EXP_NETWORK = vgg16
```

```bash
cd attack\source\ground_truth
make init（原仓库代码中没有此步骤，运行时会报错如下图）
make ground_truth
```

![image-20250301161845252](C:\Users\MIE\Desktop\主仓库\图片\image-20250301161845252.png)

