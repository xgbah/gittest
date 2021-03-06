# 大体结构

Yolo采用卷积网络来提取特征，然后使用全连接层来得到预测值。

**卷积层：**主要使用1x1卷积来做channle reduction，然后紧跟3x3卷积。

卷积层和全连接层，采用Leaky ReLU激活函数： ![[公式]](https://www.zhihu.com/equation?tex=max%28x%2C+0.1x%29) 

但是最后一层却采用线性激活函数。

## 直接上图

![img](https://pic4.zhimg.com/80/v2-5d099287b1237fa975b1c19bacdfc07f_720w.jpg)

这里我没看懂最后一个为什么从一条又变成了一块。

## 输出的具体解释

![preview](https://pic1.zhimg.com/v2-8630f8d3dbe3634f124eaf82f222ca94_r.jpg)

# 关于训练

Yolo算法将目标检测看成回归问题，所以采用的是均方差损失函数。

## 权重部分

对不同的部分采用了不同的权重值。

首先区分定位误差和分类误差。

#### 定位误差

即边界框坐标预测误差，采用较大的权重 ![[公式]](https://www.zhihu.com/equation?tex=%5Clambda+_%7Bcoord%7D%3D5) 。

#### 分类误差

然后其区分不包含目标的边界框与含有目标的边界框的置信度，对于前者，采用较小的权重值 ![[公式]](https://www.zhihu.com/equation?tex=%5Clambda+_%7Bnoobj%7D%3D0.5) ，其它权重值均设为1。

## 误差部分

采用均方误差，其同等对待大小不同的边界框，

但是实际上较小的边界框的坐标误差应该要比较大的边界框要更敏感。为了保证这一点，将网络的边界框的宽与高预测改为对其平方根的预测，即预测值变为 ![[公式]](https://www.zhihu.com/equation?tex=%28x%2Cy%2C%5Csqrt%7Bw%7D%2C+%5Csqrt%7Bh%7D%29) 。

## 另外

由于每个单元格预测多个边界框。但是其对应类别只有一个。那么在训练时，如果该单元格内确实存在目标，那么只选择与ground truth的IOU最大的那个边界框来负责预测该目标，而其它边界框认为不存在目标。

#### 好处

这样设置的一个结果将会使一个单元格对应的边界框更加专业化，其可以分别适用不同大小，不同高宽比的目标，从而提升模型性能。

#### 缺点

如果一个单元格内存在多个目标怎么办，其实这时候Yolo算法就只能选择其中一个来训练

**注意：**对于不存在对应目标的边界框，其误差项就是只有置信度，坐标项误差是没法计算的。而只有当一个单元格内确实存在目标时，才计算分类误差项，否则该项也是无法计算的。

### 最终的损失函数计算如下：

![img](https://pic3.zhimg.com/80/v2-45795a63cdbaac8c05d875dfb6fcfb5a_720w.jpg)

# 关于预测

这里我们不考虑batch，认为只是预测一张输入图片。根据前面的分析，最终的网络输出是 ![[公式]](https://www.zhihu.com/equation?tex=7%5Ctimes+7+%5Ctimes+30) ，但是我们可以将其分割成三个部分

类别概率部分为 ![[公式]](https://www.zhihu.com/equation?tex=%5B7%2C+7%2C+20%5D) 

置信度部分为 ![[公式]](https://www.zhihu.com/equation?tex=%5B7%2C7%2C2%5D) 

置信度部分为 ![[公式]](https://www.zhihu.com/equation?tex=%5B7%2C7%2C2%5D) 

然后将前两项相乘可以得到类别置信度值为 ![[公式]](https://www.zhihu.com/equation?tex=%5B7%2C+7%2C2%2C20%5D)

这里总共预测了 ![[公式]](https://www.zhihu.com/equation?tex=7%2A7%2A2%3D98) 个边界框。

## 有两种处理方法

#### 第一种策略

首先，对于**每个预测框**根据类别置信度选取**置信度最大**的那个类别作为其预测标签

经过这层处理我们得到各个预测框的预测类别及对应的置信度值，其大小都是 ![[公式]](https://www.zhihu.com/equation?tex=%5B7%2C7%2C2%5D) 。

一般情况下，会**设置置信度阈值**，就是**将置信度小于该阈值的box过滤掉**，所以经过这层处理，**剩余**的是**置信度比较高**的预测框。

最后再对这些预测框使用**NMS算法**。

#### 第二种策略

先使用NMS，然后再确定各个box的类别，

对于98个boxes，首先将小于置信度阈值的值归0，然后分类别地对置信度值采用NMS，这里NMS处理结果不是剔除，而是将其置信度值归为0。最后才是确定各个box的类别，当其置信度值不为0时才做出检测结果输出。

**我读到的文章说：**这个策略不是很直接，但是貌似Yolo源码就是这样做的。Yolo论文里面说NMS算法对Yolo的性能是影响很大的，所以可能这种策略对Yolo更好。但是我测试了普通的图片检测，两种策略结果是一样的。
