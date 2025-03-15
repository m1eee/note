1. cd infras

2. python prepare_dataset.py --dataset cifar10 数据集预处理

3. python train.py --network vgg16 --dataset gtsrb 训练模型

   ![image-20250301161404136](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250301161404136.png)

4. python export_model.py --network vgg16 --dataset gtsrb  导出训练结果/模型参数



![image-20250301161818426](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250301161818426.png)



目前遇到的问题：![image-20250301161845252](C:\Users\MIE\AppData\Roaming\Typora\typora-user-images\image-20250301161845252.png)