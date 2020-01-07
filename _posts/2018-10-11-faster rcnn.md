经过R-CNN和Fast RCNN的积淀，Ross B. Girshick在2016年提出了新的Faster RCNN，在结构上，Faster RCNN已经将特征抽取(feature extraction)，proposal提取，bounding box regression(rect refine)，classification都整合到一个网络中。

## 网络整体结构
![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-jiegou.jpg)

Faster RCNN主要分为以下4个内容：
- **Conv layer**：作为一种网络目标检测方法，Faster RCNN首先用一组基础的conv+relu+pooling层提取image的feature maps。该feature maps被共享用于后续RNP层和全连接层。
- **Region Proposal Networks(RPN网络)**：RPN网络用于产生region proposals。该层通过softmax判断anchors属于背景还是前景，再利用bounding box regression修正anchors获取精确的proposals。
- **Roi Pooling**：该层收集输入的feature maps和proposals，综合这些信息后提取proposal feature maps，送入后续全连接层判定目标类别。
- **Classification**：利用proposal feature maps计算proposal的类别，同时再次bounding box regression获取精准的检测框的位置。

下图展示了python版本中VGG16模型中的faster_rcnn_test.py的网络结构，可以清晰的看到该网络结构对一个任意大小的`P*Q`的图像，首先缩放至固定大小`M*N`，然后将`M*N`图像送入网络；而conv layer中包含了13conv层+13relu层+4pooling层；RPN网络首先经过`3*3`卷积，再分别生成前景anchors和bounding box regression偏移量，然后计算出proposals；而Roi Pooling层则利用proposals从feature maps中提取proposal feature送入后续全连接和softmax网络作classification(即判断该proposal到底是什么object)。

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-vgg16.jpg)

## 1 Conv layers
conv layer包含了conv、pooling、relu三中层。以python版本中的VGG16为例，Conv layers部分共有13个conv层，13个relu层，4个pooling层。这里有一个非常容易被忽略但是又无比重要的信息，在Conv layers中：
- 所有的conv层都是：`kernel_size=3, pad=1, stride=1`
- 所有的pooling层都是：`kernel_size=2, pad=0, stride=2`
  为何重要？在faster rcnn的conv layers中对所有的卷积都做了扩边处理(pad=1,即填充一圈0)，导致原图变为`(M+2)*(N+2)`大小，再做`3*3`卷积后输出`M*N`。正是这种设置，导致conv layers中的conv层不改变输入和输出矩阵的大小。如图所示：

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-conv.jpg)

类似的是，Conv layers中的pooling层的`kernel_size=2, pad=0, stride=2`，这样经过pooling层的`M*N`矩阵，都会变成`(M/2)*(N/2)`大小。综上所述，整个conv layers中，conv层和relu层不会改变输入输出大小，只有pooling层使输出长宽减半。

## 2 Region Proposal Networks(RPN)
经典的检测方法生成检测框都非常耗时，如OpenCV adaboost使用滑动窗口+图像金字塔生成检测框；或如R-CNN使用SS(Selective Search)方法生成检测框。而Faster RCNN则抛弃了传统的滑动窗口和SS方法，直接使用RPN生成检测框，这也是Faster R-CNN的巨大优势，能极大提升检测框的生成速度。

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-rpn.jpg)

上图展示了RPN网络的具体结构。可以看到RPN网络实际分为两条线，上面一条通过softmax分类anchors获得前景和背景。下面一条是用于计算对于anchor的bounding box regression偏移量，以便获取精准的proposal。而最后的Proposal层则负责综合前景anchors和bounding box regression偏移量获取proposals，同时剔除太小和超出边界的proposals。其实整个网络到Proposal layer这里，就已经完成了目标定位的功能。

### 2.1 多通道图像卷积知识介绍
对于多通道图像+多卷积核做卷积，计算方式如图所示：

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-multichannel.jpg)

输入有3个通道，同时有2个卷积核。对于每个卷积核，先在输入3个通道分别作卷积，再将3个通道结果加起来得到卷积输出。所以对于某个卷积层，无论输入图像有多少个通道，输出图像通道数总是等于卷积核数量！
对多通道图像做1x1卷积，其实就是将输入图像于每个通道乘以卷积系数后加在一起，即相当于把原图像中本来各个独立的通道“联通”在了一起。

### 2.2 anchors
提到RPN网络，就不能不说anchors。所谓anchors，实际上就是一组由rpn/generate_anchors.py生成的矩形。其实每行的4个值`(x_1,y_1,x_2,y_2)`表示矩形的左上角和右下角坐标。9个矩形共有3种形状，长宽比有`{1:1,1:2,2:1}`,如图所示：

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-anchor.jpg)

那么这9个anchors是做什么的呢？如下图，遍历conv layer计算获得的feature maps，为每一个点都配备这9种anchors作为初始的检测框。这样做获得检测框很不准确，不用担心，后面还有2次bounding box regression可以修正检测框位置。

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-anchors9.jpg)

解释一下上面这张图的数字。

- 在原文中使用的是ZF model中，其Conv Layers中最后的conv5层num_output=256，对应生成256张特征图，所以相当于feature map每个点都是256-dimensions
- 在conv5之后，做了rpn_conv/3x3卷积且num_output=256，相当于每个点又融合了周围3x3的空间信息（猜测这样做也许更鲁棒？反正我没测试），同时256-d不变（如图4和图7中的红框）
- 假设在conv5 feature map中每个点上有k个anchor（默认k=9），而每个anhcor要分foreground和background，所以每个点由256d feature转化为cls=2k scores；而每个anchor都有[x, y, w, h]对应4个偏移量，所以reg=4k coordinates
- 补充一点，全部anchors拿去训练太多了，训练程序会在合适的anchors中随机选取128个postive anchors+128个negative anchors进行训练

**其实，RPN最终就是在原图的尺度上，设置了密密麻麻的候选anchor。然后用cnn去判断哪些anchor里边有目标的前景anchor，哪些是没有目标的背景anchor，所以仅仅是个二分类而已！**

那么一共有多少个anchors呢？原图`800*600`，VGG下采样16倍，feature map每个点设置9个anchor，所以`ceil(800/16)*ceil(600/16)*9=17100`

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-anchorsimage.jpg)

### 2.3 softmax判定前景和背景
一个`M*N`大小的矩阵送入faster RCNN网络之后，到RPN网络变为`(M/16)*(N/16)`,不妨设`W=M/16,H=N/16`。再进入reshape和softmax层之前，先做了`1*1`卷积，如图：

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-softmx.jpg)

该1x1卷积的caffe prototxt定义如下：
```json
    layer {
      name: "rpn_cls_score"
      type: "Convolution"
      bottom: "rpn/output"
      top: "rpn_cls_score"
      convolution_param {
        num_output: 18   # 2(bg/fg) * 9(anchors)
        kernel_size: 1 pad: 0 stride: 1
      }
    }
```
可以看到其num_output=18，也就是经过该卷积的输出图像为WxHx18大小（注意第二章开头提到的卷积计算方式）。这也就刚好对应了feature maps每一个点都有9个anchors，同时每个anchors又有可能是foreground和background，所有这些信息都保存WxHx(9*2)大小的矩阵。为何这样做？后面接softmax分类获得foreground anchors，也就相当于初步提取了检测目标候选区域box（一般认为目标在foreground anchors中）。

为什么要在softmax层前后都加一个reshape层呢？其实只是为了便于softmax分类，至于具体原因这就要从caffe的实现形式说起了。在caffe基本数据结构blob中以如下形式保存数据：
```java
    blob=[batch_size, channel，height，width]
```
对应至上面的保存bg/fg anchors的矩阵，其在caffe blob中的存储形式为[1, 2x9, H, W]。而在softmax分类时需要进行fg/bg二分类，所以reshape layer会将其变为[1, 2, 9xH, W]大小，即单独“腾空”出来一个维度以便softmax分类，之后再reshape回复原状[1, 2x9, H, W]。

综上所述：RPN网络利用anchors和softmax初步提取出前景anchors作为候选区域。

### 2.4 bounding box regression原理
这一部分见[R-CNN.md](https://note.youdao.com/web/#/file/recent/markdown/3AE1F4B289DD411EA0B75F069CA5F6DE/)。

### 2.5 对proposals进行bounding box regression
在了解了bounding box regression后，再回过头来看RPN网络的第二条线路，如图所示：

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-bbox.jpg)

先来看一看上图11中1x1卷积的caffe prototxt定义：
```json
    layer {
      name: "rpn_bbox_pred"
      type: "Convolution"
      bottom: "rpn/output"
      top: "rpn_bbox_pred"
      convolution_param {
        num_output: 36   # 4 * 9(anchors)
        kernel_size: 1 pad: 0 stride: 1
      }
    }
```
可以看到其 num_output=36，即经过该卷积输出图像为WxHx36，在caffe blob存储为[1, 4x9, H, W]，这里相当于feature maps每个点都有9个anchors，每个anchors又都有4个用于回归的`[d_{x}(A),d_{y}(A),d_{w}(A),d_{h}(A)]`变换量。

### 2.6 Proposal Layer
Proposal Layer负责综合所有的`[d_{x}(A),d_{y}(A),d_{w}(A),d_{h}(A)]`变换量和前景anchors，计算出精确的proposal，送入后续ROI pooling layer。还是先来看看Proposal Layer的caffe prototxt定义：
```json
    layer {
      name: 'proposal'
      type: 'Python'
      bottom: 'rpn_cls_prob_reshape'
      bottom: 'rpn_bbox_pred'
      bottom: 'im_info'
      top: 'rois'
      python_param {
        module: 'rpn.proposal_layer'
        layer: 'ProposalLayer'
        param_str: "'feat_stride': 16"
      }
    }
```
proposal layer有三个输入，前景/背景 anchors的分类结果rpn_cls_prob_reshape，对应的bbox reg的`[d_{x}(A),d_{y}(A),d_{w}(A),d_{h}(A)]`变换量rpn_bbox_pred,以及im_info；首先解释下im_info。对于一个任意大小的`P*Q`图像，传入faster rcnn首先经过reshape到固定`M*N`，`im\_info=[M,N,scale\_factor]`则保存了此次缩放的信息。然后经过Conv Layers，经过4次pooling变为WxH=(M/16)x(N/16)大小，其中feature_stride=16则保存了该信息，用于计算anchor偏移量。

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-proposal.jpg)

proposal layer的处理过程如下：
- 生成anchors，利用`[d_{x}(A),d_{y}(A),d_{w}(A),d_{h}(A)]`变换量对所有的anchors做bbox reg回归。
- 按照输入的前景softmax scores由大到小排序anchors，提取前pre_nms_topN个anchors，即提取修正后的foreground anchors。
- 限定超出边界的foreground anchors为图像边界。
- 剔除非常小的foreground anchors；
- 进行nonmaximum suppression (nms)
- 再次按照nms排序后的foreground softmax scores由大到小foreground anchors排序，提取前post_nms_topN结果作为proposal的输出。

到此严格意义上的检测就结束了，后续部分应该是识别了！！

**RPN网络的介绍就到这里了，总结起来：生成anchors --> softmax分类器提取foreground anchors --> bbox reg回归，修正foreground anchors --> proposal layer生成proposals**

## 3 ROI Pooling
ROI Pooling层负责收集proposals，并计算出proposals feature maps，送入后续网络。从结构图中可以看到ROI pooling层有两个输入：
- 原始的feature maps
- RPN输出的proposals boxes(各不相同)，所有ROI的`N*5`的矩阵，其中N表示ROI的数目，5的第一列表示图像的index，后四列代表左上角和右下角的坐标。

### 3.1 为何需要RoI Pooling
先来看一个问题：对于传统的CNN（如AlexNet，VGG），当网络训练好后输入的图像尺寸必须是固定值，同时网络输出也是固定大小的vector or matrix。如果输入图像大小不定，这个问题就变得比较麻烦。有2种解决办法：
- 从图像中crop一部分传入网络
- 从图像中warp成需要的大小传入网络。

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-crop-wrap.jpg)

可以看到不管哪种方法都会破坏图像的原有信息。RoI Pooling就是解决了这个问题。

### 3.2 RoI Pooling原理
分析之前先来看看RoI Pooling Layer的caffe prototxt的定义：
```java
layer {
  name: "roi_pool5"
  type: "ROIPooling"
  bottom: "conv5_3"
  bottom: "rois"
  top: "pool5"
  roi_pooling_param {
    pooled_w: 7
    pooled_h: 7
    spatial_scale: 0.0625 # 1/16
  }
}
```
ROI pooling的过程：在之前有明确提到： 是对应MxN尺度的，所以首先使用spatial_scale参数将其映射回(M/16)x(N/16)大小的feature maps尺度；之后将每个proposal水平和竖直分为pooled_w和pooled_h份，对每一份都进行max pooling处理。这样处理后，即使大小不同的proposal，输出结果都是 大小，实现了fixed-length output（固定长度输出）。

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-roipooling.jpg)

ROI pooling的具体操作如下：
>- 根据输入的image，将ROI映射到feature map上的对应位置 
>  -将映射后的区域划分成大小相同的sections(sections的数量和输出的维度相同)
>- 对每个sections进行max pooling操作

***一个例子：考虑一个大小`8*8`大小的feature map，一个ROI，以及输出大小为`2*2`***

<1>. 输入固定大小的feature map

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-roi1.jpg)

<2>. region proposal投影之后位置(左上角，右下角坐标)：(0,3),(7,8)

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-roi2.jpg)

<3>. 将其划分为`(2*2)`个sections(和输出维度相同)，因此有

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-roi3.jpg)

<4>. 对每个section做max pooling，可以得到：

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-roi4.jpg)
​    

**综上所述：根据proposal的在原图中的坐标位置，并且映射到feature map中，在feature map中标出该proposal的位置。如果输出的大小是`3*3`，那么就把特征图中的ROI区域分成`3*3`模块sections，然后在每个section上取max，得到最终的`3*3`输出**。

## 4 Classification
Classification利用已经获得的proposal feature maps，通过full connect层与softmax层计算每个proposal具体属于哪个类别(人、车等),输出cls_prob概率向量；同时再次利用bounding box regression获取每个proposal的位置偏移量bbox_pred，用于回归更加精确的目标检测框。
Classification的部分网络结构图：

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-classification.jpg)

从Pol Pooling获取到7X7=49大小的proposal feature maps后，送入后续网络，可以看到做了如下两件事：
- 通过全连接和softmax对proposal进行分类，这实际上已经是识别的范畴了；
- 再次对proposals进行bounding box regression，获取更高精度的rect box；

## 5 Faster RCNN网络训练
Faster R-CNN的训练，是在已经训练好的model（如VGG_CNN_M_1024，VGG，ZF）的基础上继续进行训练。实际中训练过程分为6个步骤：
1. 在已经训练好的model上，训练RPN网络；
2. 利用步骤1中训练好的RPN网络，收集proposals；
3. 第一次训练Fast RCNN网络；
4. 第二次训练RPN网络；
5. 再次利用步骤4中训练好的RPN网络，收集proposals；
6. 第二次训练Fast RCNN网络。
  下面是一张训练过程的流程图，应该更加清晰。

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-training.jpg)

### 5.1 训练RPN网络
在该步骤中，首先读取RBG提供的预训练好的model(本文使用VGG)，开始迭代训练，网络结构如图所示：

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-rpntrain.jpg)

与检测网络类似的是，依然使用Conv layers提取feature maps。整个网络的Loss如下：
```math
    Loss= \frac{1}{N_{cls}}\sum_i{L_{cls}(p_i,pi^*)}+\lambda \frac{1}{N_{reg}}\sum_i{p_i^*L_{reg}(t_i,t_i^*)}
```
上述公式中，i表示anchors index，`p_i`表示foreground softmax probability，`p_i^*`表示对应的gt prdict的概率(注意`0.3<IOU<0.7`的anchor不参与训练)；t代表predict bounding box，`t^*`代表对应foreground anchor对应的gt box。可以看到整个loss分为两个部分：
1. **classification loss**，即rpn_cls_loss层计算的softmax loss，用于分类anchors为foreground与background的网络训练。
2. **regression loss**，即rpn_loss_bbox层计算的smooth L1 loss，用于bounding box regression网络训练。注意到该loss中乘了`p_i^*`，相当于只关心foreground anchors的回归。

常用的smooth L1 loss损失的计算公式如下：

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/smooth.png)

### 5.2 通过训练好的RPN网络收集proposals
在该步骤中，利用之前的RPN网络，获取proposals rois，同时获取foreground softmax probability，网络如下图所示：

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-proposalstrain.jpg)

### 5.3 训练Faster RCNN网络
读取之前保存的pickle文件，获取proposals与foreground softmax probability。从数据层输入网络，然后：
- 将提取的proposals作为rois传入网络，如蓝框所示；
- 计算bbox_inside_weights+bbox_outside_weights，作用与RPN一样，传入smooth_L1_loss_layer，如绿框所示；

这样就可以训练最后的识别softmax与最终的bounding box regression了。

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/faster-rcnn-train-fastrcnn.jpg)