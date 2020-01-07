## R-CNN、SPP-Net、Fast R-CNN、Faster R-CNN的关系

RCNN网络的演变过程如图所示：

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/%E4%B8%89%E8%80%85%E6%AF%94%E8%BE%83-%E7%89%A9%E4%BD%93%E8%AF%86%E5%88%AB%E7%AE%97%E6%B3%95.png)

算法名称 | 使用方法 | 缺点 | 改进
---|---|---|---
R-CNN | (1).SS提取RP; (2).CNN提取特征；(3).SVM分类；(4).BB盒回归 | (1).训练步骤繁琐(微调网络+训练SVM+训练bbox)；(2).训练、测试速度都很慢；(3).训练占空间太大; (4).**每个区域都要进行CNN网络计算，浪费时间** | (1).从DPM HSC的34.3%提升到66%；(2).引入RP+CNN 
SPP-Net | (1).SS提取RP; (2).CNN提取特征；(3).SVM分类；(4).BB盒回归 | (1).和RCNN一样，训练过程仍是分离的，需多次转存; (2).SPP-Net无法同时微调在SPP-Net层两边的fc层和conv层; | **(1).取消了归一化的过程，解决了因变形信息丢失的问题; (2).采用“空间金字塔池化”替换了全连接层之前的最后一个池化层**
Fast R-CNN | (1).SS提取RP；(2).CNN提取特征；**(3).softmax分类；(4).多任务损失函数边框回归** | (1).依旧使用SS提取RP(耗时2~3秒，特征提取耗时0.32秒)；(2).无法满足实时应用，不能实现端到端训练测试；(3).利用了GPU,但是区域提取方法是在CPU中进行的; (4).候选框的提取RP仍然比较耗时较大 | (1).由66.9%提升到70%；(2).每张图像耗时约3秒; **(3).提出简化版的ROI池化层; (4).改SVM为softmax;(5).采用多任务loss层训练bbox**
Faster R-CNN | **(1).RPN提取RP**；(2).CNN提取特征; (3).softmax分类; (4).多任务损失函数边框回归 | (1).还是无法达到实时监测目标; (2).获取RP,再对每个候选proposal分类计算量还是比较大，**速度仍需提升** | (1).提高了检测进度和速度; (2).真正实现了端到端的目标检测框架; **(3).引入RPN网络机制，生成RP仅需要10ms**


## 几种物体识别算法原理图
### R-CNN原理
![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/RCNN%E5%8E%9F%E7%90%86.png)

### SPP-Net原理
![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/SPP-Net%E5%8E%9F%E7%90%86.png)

### Fast R-CNN

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/Fast%20RCNN%E5%8E%9F%E7%90%86-%E6%B5%81%E7%A8%8B%E7%89%88.png)

### Faster R-CNN

![image](https://raw.githubusercontent.com/liurio/deep_learning/master/img/Faster%20RCNN%E5%8E%9F%E7%90%86-RPN.png)