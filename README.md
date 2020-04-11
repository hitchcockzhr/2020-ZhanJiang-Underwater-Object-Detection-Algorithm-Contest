# 2020-ZhanJiang-Underwater-Object-Detection-Algorithm-Contest
2020湛江水下目标检测算法赛

## 前言
此次比赛分数不是很理想，A榜20.14，B榜19.63。我认为的原因应该有很多，比如说模型，水下图像增强算法，trick，
又或者是评价指标比较苛刻（赛事采用的是COCO mAP[@0.5:0.05:0.95] 指标）等等，具体的原因会在之后的分部分详细解释。当然，这也是我第一次参加目标检测的比赛，
所以过程中难免会有疏漏。而因为此次比赛，让我对keras非常失望，
具体原因可见我的这一个blog:[Choice keras or pytorch?](https://fieldhunter.github.io/talking_about_keras_and_pytorch/)。虽然说分数不理想，
但也是好歹做了出来，自己也有自己的idea在里面，只不过应该是中途哪里出了问题，我没发现罢了（但我真的看了源码很久觉得应该没问题），
可能换了一些策略或者单换个PyTorch实现的模型之后分数就会大幅上涨，毕竟大佬提供的PyTorch的baseline能达到挺高的分数。但肯定的是自己的idea是挺有效的，
**从裸模型+图像增强的10分提高到了20分**。（所以我就怀疑是yolo的问题）所以在此将它放入我的github中，对今后参加类似的比赛或者项目肯定有借鉴意义。
(租GPU就花了500块，真的吐血)

## 说明
没有上传的文件夹有data、models和pre_train，即文件太大，不宜上传。原data目录当中，test目录下是测试集，包括图像增强后的图像，trian目录下是训练集，
也包括图像增强后的图像。data目录中还有yolo模型所需的classes.txt、yolo_anchors.txt和train_data.txt。在pre_train目录当中，有yolo的预训练模型yolov3.weight，
以及用convert.py转换的yolo_weights.h5。在models目录中，存储训练结束的模型和Tensorboard记录。

## 图像增强
首先，对图像采取增强算法增强(image_aug.py)后进行存储。对于水下图像识别这个领域，我也是参加这个比赛才接触的，同样是第一次参加目标检测的比赛，
所以着重的去对模型算法做更改而不是针对数据，所以图像增强的算法只是照搬了赛事群里的一个大佬的开源，可视化出来后看着效果是挺不错的，
但具体怎么样我也不敢下定论，所以可能水下图像增强算法的效果不是很好，导致分数不是很高。具体的算法在data_augmention目录里。为了满足后续的数据增强需求，
**对训练数据进行原图尺寸增强**，但会使得处理速度很慢。为了解决这个问题，尝试采用gpu版的numpy:cupy来调用gpu加速处理，
但增强算法中的有些numpy函数cupy尚未支持，所以只能用numpy，之后采用了10个进程的多进程方式来尽量的加速处理。**对于测试集，直接缩放至目标尺寸**来增强，
这样处理速度就会非常快，避免大尺寸图片耗时的问题，之后再进行输入处理，以满足模型输入需求。具体的处理方式在[数据增强](#数据增强)部分介绍。

另外，在使用大佬的开源增强算法之前，使用过两个UnderWaterGAN，但是他们的模型输出都是256\*256像素，整张图片和目标都扭曲的不成样子，效果还不是特别的好，其中一个还非常的差。所以就放弃使用对抗生成网络。（可能图像的尺寸和GAN网络本身训练的不符，强行将几千的像素压缩到256，因为看那些README效果都很不错）

## 数据预处理
data_process.py中，对所有图像进行统计，对每一个候选框，进行细致的处理，代码中给出了详细的注释，这里不再阐述。
候选框处理时**删除那些面积小于120的过小候选框**。之后再靠kmeans.py来生成yolo所需的anchors，最后得到data目录中的train_data.txt和yolo_anchors.txt。

另外，赛事的目标种类只有4种，但数据集里有5种，多了一个水草。观察数据集后发现，水草和海胆在某些情况下十分相似，为了使模型增强对水草的区分能力，
**在训练时用5个类别去训练，在预测的时候把水草类丢弃即可**。

## 数据增强
数据增强代码在yolo3目录的utils.py中。对于模型的输入尺寸，**采用480*480**，原yolo尺寸是416\*416。经过实验验证，**从416到448再到480，分数都是稳步提高的**。
由于时间和成本的关系，并没有尝试512\*512尺寸，这或许会进一步提高。针对目标检测的task，并不希望把任意尺寸的图片强行resize，策略是选出高与宽较长的一边，
计算这条边缩放到480的比例，然后按照这个比例对原图进行**等比例缩放**。等比缩放后，长的一边是480，短的一边则不是。
之后在短边上**用灰色像素来填充**以满足480\*480尺寸。统计过数据集的尺寸，都是长大于高，所以短边都是高这条边。灰色像素数量在缩放后的原图上下两侧均分，
这样使得处理后的图像**正中间是缩放后的训练图像，上下两侧是灰色像素**。[图像增强](#图像增强)部分的测试集处理也是如此。ps:此处尝试用base64来导入图片演示，
但根本加载不出来，不知道为什么，只能这样靠着文字和代码来理解一下。

具体步骤：对于训练数据，首先按照7:3的可能性进行**直接缩放或者随机裁剪**。直接缩放过程中删除缩放后面积小于120的过小候选框，
随机裁剪出来的尺寸也与按照原图直接缩放策略缩放后的尺寸相同。之后都是按照1:1的可能性**按顺序进行二次的轻度图像增强，水平翻转和垂直翻转**。对于测试集，
直接进行缩放即可。之后按照上述所说的策略填充灰色像素。最后进行模型输入格式的处理，**像素归一化和候选框处理**。同样，代码中有详细的注释。

## 模型训练
在train.py中，输入尺寸为480*480，训练集和测试集按照9:1划分。训练过程分为两个阶段来训练，第一阶段**冻结预训练所有层**，采用RAdam(最小值设为1e-5)，
warm_up策略，batch_size设为32，训练100轮，同时做Tensorboard记录。在第二阶段**打开全部网络层来训练**，采用RAdam(最小值设为1e-6)，warm_up策略，
**swa算法，cosine-annealing学习率策略（范围1e-2到1e-6），batch_size设为8（主要原因是显存限制）**，训练200轮，同时做Tensorboard记录，
通过ModelCheckpoint策略对每一轮按照val_loss来决定是否保存模型，最终选用val_loss最小的模型来做预测。即将最好的模型更名为trained_weights_final.h5。

在模型设计上(yolo3目录中的model.py)，对class_loss进行修改，采用**label_smoothing策略**。由于在预测的可视化结果中看到候选框的及分类的结果都挺不错，
但是一些很明显且已经识别出来的目标的置顶度却很低，一些显然不是目标的置信度相对的有点高，所以想对confidence_loss对更改，
即给它整体或者正样本部分(confidence_loss分为正样本和负样本两部分)乘上一个值，让模型更注重置信度的损失，简单粗暴的想法。但结果并不是很好，
在训练过程中模型的loss也相对于之前高了几十个点并且下不去，最终的分数也比之前下降了一些。加上COCO mAP[@0.5:0.05:0.95] 这个评价指标候选框越多越好，
所以最终放弃了对confidence_loss的修改。

对于其他模型，比如Faster-RCNN，我也有考虑去用过。并且听赛事群里说比赛基本没人用单阶段模型，想想也是，二阶段模型肯定准确率会高一些。但是......算了，
不想吐槽keras了，至于为什么没有用除了yolo以外的模型，还是看看[前言](#前言)里我提到的我写的blog吧。

## 预测
在predict.py中，对预测出来的候选框进行**WBF**，相对于之前使用的NMS，WBF策略**提升了不少分数**。之后就是一系列的后处理操作，代码中的注释很详细，
这里不再阐述。另外，**赛事数据集的候选框坐标起点是1，在训练前已经更改为0，所以在后处理中需要再次更改为1**。

在参数上，**score阈值设为0.001，iou阈值（包括模型阈值和WBF阈值）设为0.25**能得到最高的分数。

## ps
**数据预处理和数据可视化**都有Jupyter版本。分别进行了**训练集的统计信息**工作和**各阶段数据（训练集，数据增强，预测结果）可视化**工作。同样也有详细的注释。

最终训练出来最好的模型和Tensorboard记录我会存放在releases当中。

yolov3部分的代码是基于[qqwweee/keras-yolo3](https://github.com/qqwweee/keras-yolo3)进行更改的。
