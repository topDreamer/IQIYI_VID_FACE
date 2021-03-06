# IQIYI多模态视频人物识别
## 队伍：炸天

---
## 运行环境
基于官方特征向量的人脸识别: python3.6+keras

场景识别: python2.6+mxnet

GPU：Nvidia P100

## 模型摘要
### 基于特征向量的人脸识别
运用Dropout、Batchnormalization、负样本处理以及**对特征向量进行数据增强**，单模型效果可达0.8259。
融合多模型后分数可达0.8381。
### 基于场景的人物识别
每个视频取两帧图片进行训练。为了兼顾训练速度和准备率，选择模型为[resnext](https://github.com/bruinxiong/SENet.mxnet)。 对测试集中人脸质量较差或检测不到人脸的视频进行识别，并将结果跟基于特征向量得到的结果进行融合。 融合后分数可达0.8505（最终结果）。

## 算法流程说明
### 预处理
#### preprocess/get_data_mean.py
对特征向量进行数据增强。 提供的特征向量是基于arcface模型训练的，因为不同人物对应的特征向量之间存在着几何意义，即一个人物的特征向量的集合是一个闭集。 集合中任意N个向量的平均值仍在集合中。

我们对每个视频中特征向量进行了增强，随机取2,3,4,5个特征向量，求出其平均值。将得到的新特征向量加入到原始数据中。

基于该方法，我们将测试集的特征向量的数量扩充了接近一倍。

但该方法对训练数据的扩充效果并不明显，因为扩充的数据仍在集合内，集合的范围没有因此扩大或者缩小，对于训练模型而言帮助有限。因此我们只对测试数据进行了数据增强。


#### preprocess/get_test_img_list.py
得到测试集中，人脸质量较低（低于40）或者检测不到人脸的视频。
并将其分别记录。这部分数据是用于基于场景的人物识别模型的。

### 基于特征向量的人脸识别
#### 训练阶段  simple_model/train_ini.py
运行脚本 simple_model/train.sh

验证集和训练集都用来训练，将验证集中的噪声数据设为第4935类。每次训练提取全部数据的80%。

每次训练包括4个模型，分别取人脸质量为0到200,20到200,40到200,0到80的数据进行训练。
一共训练18次，通过控制random seed，使得每次训练的数据不同。总共可得到72个模型。模型保存在data/simple_model/save_model/中

#### 预测阶段 simple_model/predict.py
运行脚本  simple_model/pre.sh

每次预测同一次训练的4个模型（机器内存限制），然后将得到的预测结果取平均。总共预测18次，得到18个结果。再将得到的结果，每6个合并为一个。全部得到3个结果，保存在data/simple_model/ 中。
将得到的三个结果，与我之前瞎调得到的最优结果（分数为0.8282）融合（数据保存在data/simple_model/tmp_result中）。得到的结果即为我们队伍基于特征向量的最优结果，分数为0.8381。

分次预测及分次融合是因为机器的限制，若机器内存足够，可一次完成。
预测结果融合的代码及脚本为merge.py、merge.sh


### 基于场景的人物识别
#### 数据准备
训练集：每个视频取第一帧和最后一帧。使用mxnet官方方法生产.rec文件，resize和crop成224*224的大小。 此处不赘述。

测试集： 我们将人脸识别为0-20（包含无人脸）、20-30、30-40分别存放在3个文件夹中。

预训练模型存放在data/resnext_model/model中

#### 训练 resnext/train_se_resnext_w_d
运行脚本： resnext/run.sh

使用基于imagenet的预训练模型。训练16个epoch。训练好的模型保存在data/resnext_model/save_model中


#### 预测
运行脚本： resnext/predict.sh

对每个视频的所有帧进行预测，对所有帧的预测结果取平均，作为该视频的预测结果。
#### 融合  resnext/merge.py
将resnext得到的预测结果与基于特征向量得到的结果做融合。

人脸质量为0到20的，特征向量的结果权重为0.5，resnext得到的结果权重为0.5。人脸质量为320-30的，权重分别为0.6、0.4。人脸质量为30-40的，权重分别为0.7、0.3。

对于那些检测不到人脸的视频，只取resnext得到的结果，权重为0.8.（因为resnext得到的结果性能并不优越，在验证集上约有46%的准确率）
最终提交的结果保存在submit_result/best_merge.pickle中