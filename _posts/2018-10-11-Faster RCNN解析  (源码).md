### main()
网络以VGG-16网络为例，从main函数出发
- tools/train_net.py
```python
...
# 加载训练数据，主要有get_imdb()用来获取imdb，get_training_roidb(imdb)获取roidb
# roidb是属于imdb的，roidb并没有真正的读取数据，只是建立了一些相关信息，是dict
imdb, roidb = combined_roidb(args.imdb_name)
...
net = vgg16()
...
train_net(net, imdb, roidb, valroidb, output_dir, tb_dir,
            pretrained_model=args.weight,
            max_iters=args.max_iters)
```
### combined_roidb()
**tools/train_net.py/combined_roidb()**
```python
...
imdb = get_imdb(imdb_name) # factory.py在工厂类中的get_imdb()
            \\
             \\
              __sets[name] = (lambda split=split, year=year: pascal_voc(split, year)) # 生成不同的imdb
                        \\
                         \\
                          self._load_image_set_index() # 一个列表，包含对应数据集图像的名称信息，如[000001]，获取图片的索引
                          self.gt_roidb # 得到roi的图片信息，然后重载imdb中
                                \\
                                 \\
                                  self._load_pascal_annotation(index) # 返回的是图片信息dict，然后按照顺序存进一个list，
                                  #对应图片信息索引,从数据库的xml文件中加载图像和bbox等信息，包括bboxes坐标，类别，overlap矩阵，
                                  #bbox面积等
...
imdb.set_proposal_method(cfg.TRAIN.PROPOSAL_METHOD) # 读取imdb后，执行set_proposal_method函数，该函数把字符串转成表达式
roidb = get_training_roidb(imdb) # 生成roidb
            \\
             \\
              imdb.append_flipped_images() # 表示是否对图片进行翻转，图片增强，为了扩充数据库，对图片进行翻转
              rdl_roidb.prepare_roidb(imdb) # 该函数主要是用来准备imdb的roidb，主要工作是给roidb中的字典添加一些属性
```
到此combined_roidb()函数部分结束，此时并没有真正的加载数据，只是类似dict
### train_net()
**model/train_val.py/train_net()**
```python
roidb = filter_roidb(roidb) # 去除没有用的rols，要保证至少一个前景或背景
            ||
            ||
           \\//
sw.train_model(sess, max_iters)
```
**train_model()**
```python
self.data_layer = RoIDataLayer(self.roidb, self.imdb.num_classes)
                \\
                 \\
                  self.shuffle_roidb_inds() #把roidb索引打乱，重新获取index
lr, train_op = self.construct_graph(sess) #定义损失，优化器，网络等
                \\
                 \\
                 layers = self.net.create_architecture(...) # 开始构造RPN，回归等网络，并且定义loss
                            \\
                             \\
                              rois, cls_prob, bbox_pred = self._build_network(training) # 获取RPN/bbox等网络
                              self._add_losses() # 定义损失函数
                 self.optimizer = tf.train.MomentumOptimizer(lr, cfg.TRAIN.MOMENTUM) #定义优化器
                 gvs = self.optimizer.compute_gradients(loss)
                 train_op = self.optimizer.apply_gradients(gvs)

lsf, nfiles, sfiles = self.find_previous() #加载之前存在的snapshot
...
blobs = self.data_layer.forward() #用梯度下降法训练，一次获取一个batch
```

**train_model() -> construct_graph()**
```python
layers = self.net.create_architecture(...) # 开始构造RPN，回归等网络，并且定义loss
self.optimizer = tf.train.MomentumOptimizer(lr, cfg.TRAIN.MOMENTUM) #定义优化器
gvs = self.optimizer.compute_gradients(loss)
train_op = self.optimizer.apply_gradients(gvs)
# compute_gradients()和apply_gradients()加起来相当于minimize()
```
**train_model() -> construnct_graph() -> create_architecture()**
```python
...
rois, cls_prob, bbox_pred = self._build_network(training) # 获取RPN/bbox等网络
...
self._add_losses() # 定义loss,包括分类loss和回归loss
```
### _build_network()
**train_model() -> construnct_graph() -> create_architecture() -> _build_network()**
```python
initializer = tf.random_normal_initializer(mean=0.0, stddev=0.01) #全局初始化
net_conv = self._image_to_head(is_training) #如果是VGG网络，就执行vgg.py中的函数，主要负责VGG网路的构建
self._anchor_component() #生成anchors，每个网格有9个
rois = self._region_proposal(net_conv, is_training, initializer) # 构建RPN网络
...
pool5 = self._crop_pool_layer(net_conv, rois, "pool5")
fc7 = self._head_to_tail(pool5, is_training) #VGG网络的全连接层部分
cls_prob, bbox_pred = self._region_classification(fc7, is_training, initializer) #构建分类回归网络
```
#### 构建VGG-16网络(卷积层，最后一个卷积除外)
**train_model() -> construnct_graph() -> create_architecture() -> _build_network() -> vgg16/_image_to_head()**
```python
#13conv + 13relu + 4pool
net = slim.repeat(self._image, 2, slim.conv2d, 64, [3, 3], trainable=False, scope='conv1')
net = slim.max_pool2d(net, [2, 2], padding='SAME', scope='pool1')
net = slim.repeat(net, 2, slim.conv2d, 128, [3, 3], trainable=False, scope='conv2')
net = slim.max_pool2d(net, [2, 2], padding='SAME', scope='pool2')
net = slim.repeat(net, 3, slim.conv2d, 256, [3, 3], trainable=is_training, scope='conv3')
net = slim.max_pool2d(net, [2, 2], padding='SAME', scope='pool3')
net = slim.repeat(net, 3, slim.conv2d, 512, [3, 3], trainable=is_training, scope='conv4')
net = slim.max_pool2d(net, [2, 2], padding='SAME', scope='pool4')
net = slim.repeat(net, 3, slim.conv2d, 512, [3, 3], trainable=is_training, scope='conv5')
```
#### 生成anchors
**train_model() -> construnct_graph() -> create_architecture() -> _build_network() -> _anchor_component()**
```python
anchors, anchor_length = tf.py_func(generate_anchors_pre,
                                            [height, width,
                                             self._feat_stride, self._anchor_scales, self._anchor_ratios],
                                            [tf.float32, tf.int32], name="generate_anchors")

---------
generate_anchors_pre() 
                \\
                 \\
                  \\
                  layer_utils/snippets.py/generate_anchors() # 产生anchors的函数
```
**train_model() -> construnct_graph() -> create_architecture() -> _build_network() -> _anchor_component() -> generate_anchors()**
```python
def generate_anchors(base_size=16, ratios=[0.5, 1, 2],
                     scales=2 ** np.arange(3, 6)):
  """
  Generate anchor (reference) windows by enumerating aspect ratios X
  scales wrt a reference (0, 0, 15, 15) window.

  anchor的表现形式有两种，一种记录左上角和右下角的坐标，一种是记录中心坐标和宽高
  这里生成一个基准anchor，采用左上角和右下角的坐标表示方式[0,0,15,15]
  """
  base_anchor = np.array([1, 1, base_size, base_size]) - 1 #[0,0,15,15]
  ratio_anchors = _ratio_enum(base_anchor, ratios) # 返回的是不同长宽比的anchor
  anchors = np.vstack([_scale_enum(ratio_anchors[i, :], scales)
                       for i in range(ratio_anchors.shape[0])]) #生成九个候选框 shape: [9,4]
  return anchors

def _ratio_enum(anchor, ratios):
  """
  Enumerate a set of anchors for each aspect ratio wrt an anchor.

  计算不同长宽比尺度下的anchor坐标
  """

  w, h, x_ctr, y_ctr = _whctrs(anchor)  # 找到中心点的坐标和长宽
  size = w * h #返回anchor的面积
  size_ratios = size / ratios  #为了计算anchor的长宽尺度设置的数组：array([512.,256.,128.])
  ws = np.round(np.sqrt(size_ratios))  #计算不同长宽比下的anchor的宽：array([23.,16.,11.])
  hs = np.round(ws * ratios) #计算不同长宽比下的anchor的长 array([12.,16.,22.])
  anchors = _mkanchors(ws, hs, x_ctr, y_ctr) #返回新的不同长宽比的anchor，返回的数组shape[3,4]，anchor是左上角和右下角的坐标
  return anchors 


def _scale_enum(anchor, scales):
  """
  Enumerate a set of anchors for each scale wrt an anchor.

  这个函数对于每一种长宽比的anchor，计算不同面积尺度的anchor坐标
  """

  w, h, x_ctr, y_ctr = _whctrs(anchor) #找到anchor的中心坐标
  ws = w * scales #shape [3,] 得到不同尺度的新的宽
  hs = h * scales #shape [3,] 得到不同尺度的新的高
  anchors = _mkanchors(ws, hs, x_ctr, y_ctr)
  return anchors
```
#### RPN网络
**train_model() -> construnct_graph() -> create_architecture() -> _build_network() -> _region_proposal()**
```python
#进入RPN网络，先进行3x3卷积
rpn = slim.conv2d(net_conv, cfg.RPN_CHANNELS, [3,3], trainable=is_training, weights_initializer=initializer, scope="rpn_conv/3x3") 
"""
    开始两条线路，一条是分类，一条是回归bbox
    1：分类：1x1 -> reshape -> softmax -> reshape   
                                                   \
                                                    proposal 
                                                   /    
    2: 回归：1x1
"""
rpn_cls_score = slim.conv2d(rpn, self._num_anchors * 2, [1, 1], trainable=is_training,
                                weights_initializer=initializer,
                                padding='VALID', activation_fn=None, scope='rpn_cls_score') # 这部分是分类线路，先经过1x1卷积
rpn_cls_score_reshape = self._reshape_layer(rpn_cls_score, 2, 'rpn_cls_score_reshape')
rpn_cls_prob_reshape = self._softmax_layer(rpn_cls_score_reshape, "rpn_cls_prob_reshape") # 利用softmax对所有的anchor进行分类，选出前景
rpn_cls_pred = tf.argmax(tf.reshape(rpn_cls_score_reshape, [-1, 2]), axis=1, name="rpn_cls_pred")
rpn_cls_prob = self._reshape_layer(rpn_cls_prob_reshape, self._num_anchors * 2, "rpn_cls_prob") # 得到分类后的结果

...
# 第二条线路
 rpn_bbox_pred = slim.conv2d(rpn, self._num_anchors * 4, [1, 1], trainable=is_training,
                                weights_initializer=initializer,
                                padding='VALID', activation_fn=None, scope='rpn_bbox_pred') #回归
...                                
rois, roi_scores = self._proposal_layer(rpn_cls_prob, rpn_bbox_pred, "rois")
rpn_labels = self._anchor_target_layer(rpn_cls_score, "anchor")
rois, _ = self._proposal_target_layer(rois, roi_scores, "rpn_rois")
rois, _ = self._proposal_top_layer(rpn_cls_prob, rpn_bbox_pred, "rois")
```
**train_model() -> construnct_graph() -> create_architecture() -> _build_network() -> _region_proposal() -> _proposal_layer()**
```python
"""
    功能是生成proposal，并筛选nms等，主要流程概况为以下四点：
    1、利用坐标变换生成proposal，proposals=bbox_transform_inv(anchors, rpn_bbox_prd)
    2、按前景概率对proposal进行排序，然后留下RPN_PRE_NMS_TOP_N个proposals
    3、对剩下的proposal进行nms操作，阈值是0.7
    4、对剩下的proposal，保留RPN_POST_NMS_TOP_N个，得到最终的rois和相应的rpn_score。
"""
# 按照通道C取出RPN预测的框，属于前景的分数，请注意，这18个channel中，前9个是背景的概率，后9个是前景的概率
scores = rpn_cls_prob[:, :, :,num_anchors:] 
proposals = bbox_transform_inv(anchors, rpn_bbox_pred) # 结合RPN的输出变换坐标，得到第一次变换后的proposals
proposals = clip_boxes(proposals, im_info[:2]) # 将超出图像边界的proposal进行边界裁剪
...
"""
 按照前景概率进行排序，取前top个，
  对框按照前景分数进行排序，order中指示了框的下标
   Pick the top region proposals
"""
  order = scores.ravel().argsort()[::-1]
  if pre_nms_topN > 0:
    order = order[:pre_nms_topN] 
    
  #选择前景分数排名在前pre_nms_topN(训练时为12000，测试时为6000)的框
  proposals = proposals[order, :] #保留了前pre_nms_topN个框的坐标信息
  scores = scores[order] #保留了前pre_nms_topN个框的分数信息
  """
   对剩下的proposal进行NMS操作，阈值是0.7进行nms操作，再取前n个
   Non-maximal suppression
   使用nms算法排除重复的框
  """
keep = nms(np.hstack((proposals, scores)),nms_thresh)


# 对剩下的proposal，保留RPN_POST_NMS_TOP_N个， 得到最终的rois和相应的rpn_socre
  # Pick th top region proposals after NMS
  if post_nms_topN > 0:
    keep = keep[:post_nms_topN] #选择前景分数排名在前post_nms_topN(训练时为2000，测试时为300)的框
  proposals = proposals[keep, :] #保留了前post_nms_topN个框的坐标信息
  scores = scores[keep] #保留了前post_nms_topN个框的分数信息
```
**train_model() -> construnct_graph() -> create_architecture() -> _build_network() -> _region_proposal() -> anchor_target_layer()**
```python
"""
    该函数的作用是得到训练RPN所需要的标签，并计算anchor与gt_bbox之间的差值，即rpn_bbox_targets。
    其中正负样本的生成策略：
    1、只保留图像内部的anchors
    2、对于每个gt_box，找到与它最大的IoU的anchor，则设为正样本；
    3、对于每个anchor，与任意一个gt_box的IoU>0.7为正样本，<0.3为负样本；
    4、其他的anchor忽略
    5、加入正负样本过多，则进行采样，采样比例由RPN_BATCHSIZE，RPN_FG_FRACTION等控制
"""
anchors = all_anchors[inds_inside,:] # 仅保留图像内部的anchor
overlaps = bbox_overlaps(
    np.ascontiguousarray(anchors, dtype=np.float),
    np.ascontiguousarray(gt_boxes, dtype=np.float)) # 计算anchors和gt_boxes的IoU
    
# IoU>0.7为正样本，<0.3为负样本
if not cfg.TRAIN.RPN_CLOBBER_POSITIVES: 
    labels[max_overlaps < cfg.TRAIN.RPN_NEGATIVE_OVERLAP] = 0
  labels[gt_argmax_overlaps] = 1
  labels[max_overlaps >= cfg.TRAIN.RPN_POSITIVE_OVERLAP] = 1
  if cfg.TRAIN.RPN_CLOBBER_POSITIVES:
    labels[max_overlaps < cfg.TRAIN.RPN_NEGATIVE_OVERLAP] = 0
    
# 假如正负样本过多，则进行采样，采样比例由RPN_BATCHSIZE， RPN_FG_FRACTION等控制
  num_fg = int(cfg.TRAIN.RPN_FG_FRACTION * cfg.TRAIN.RPN_BATCHSIZE)

bbox_targets = _compute_targets(anchors, gt_boxes[argmax_overlaps,:]) # 用来计算roi和真实gt的偏移量,输出[dx,dy,dw,dh]

# bbox_inside_weights是loss函数分类的权重，bbox_outside_weights是loss函数回归的权重
bbox_inside_weights[labels == 1, :] = np.array(cfg.TRAIN.RPN_BBOX_INSIDE_WEIGHTS)
bbox_outside_weights[labels == 1, :] = positive_weights
bbox_outside_weights[labels == 0, :] = negative_weights
```
**train_model() -> construnct_graph() -> create_architecture() -> _build_network() -> _region_proposal() -> anchor_target_layer() -> _compute_targets()**
```python
return bbox_transform()

bbox_transform(): # 该函数的作用是得到anchor相对于GT的(dx,dy,dw,dh)四个回归值
      # 计算每一个anchor的width和height
      ex_widths = ex_rois[:, 2] - ex_rois[:, 0] + 1.0
      ex_heights = ex_rois[:, 3] - ex_rois[:, 1] + 1.0
      # 计算每一个anchor的中心店的坐标x,y
      ex_ctr_x = ex_rois[:, 0] + 0.5 * ex_widths
      ex_ctr_y = ex_rois[:, 1] + 0.5 * ex_heights
      
      #注意：当前的GT不是最一开始传进来的所有GT，而是与对应anchor最匹配的GT，可能有重复信息
      #计算每一个GT的width与height
      gt_widths = gt_rois[:, 2] - gt_rois[:, 0] + 1.0
      gt_heights = gt_rois[:, 3] - gt_rois[:, 1] + 1.0
      #计算每一个GT的中心点x，y坐标
      gt_ctr_x = gt_rois[:, 0] + 0.5 * gt_widths
      gt_ctr_y = gt_rois[:, 1] + 0.5 * gt_heights
"""
    要对bbox进行回归需要4个量，dx、dy、dw、dh，分别为横纵平移量、宽高缩放量
  #此回归与fast-rcnn回归不同，fast要做的是在cnn卷积完之后的特征向量进行回归，dx、dy、dw、dh都是对应与特征向量
    此时由于是对原图像可视野中的anchor进行回归，更直观

    定义 Tx=Pwdx(P)+Px Ty=Phdy(P)+Py Tw=Pwexp(dw(P)) Th=Phexp(dh(P))
    P为anchor，T为target，最后要使得T～G，G为ground-True
    回归量dx(P)，dy(P)，dw(P)，dh(P)，即dx、dy、dw、dh
"""
      targets_dx = (gt_ctr_x - ex_ctr_x) / ex_widths
      targets_dy = (gt_ctr_y - ex_ctr_y) / ex_heights
      targets_dw = np.log(gt_widths / ex_widths)
      targets_dh = np.log(gt_heights / ex_heights)
  
     #targets_dx, targets_dy, targets_dw, targets_dh都为（anchors.shape[0]，）大小
    #所以targets为（anchors.shape[0]，4）
     targets = np.vstack(
     (targets_dx, targets_dy, targets_dw, targets_dh)).transpose()
    return targets
```
**train_model() -> construnct_graph() -> create_architecture() -> _build_network() -> _region_proposal() -> proposal_target_layer()**
```python
"""
    该函数的作用是：为上一步得到的proposal分配的所属物体类别，并得到proposal和gt_bbox的坐标位置间的差别，以便下一步的分类和回归
    主要步骤：
    1、确定每张图片的roi的数目，以及前景fg_roi的数目
    2、从所有的rpn_rois进行采样，并得到rois的类别以及bbox的回归坐标(bbox_targets)，即真实值与预测值之间的偏差。
"""
all_rois = rpn_rois # 表示选择出来的n个待进行分类的框以及score
# 将gt框加入到待分类的框内，相当于增加了正样本的个数
all_rois = np.vstack(
      (all_rois, np.hstack((zeros, gt_boxes[:, :-1]))) #all_rois输出维度(N+M,5)，前一维表示是从RPN的输出选出的框和ground truth框合在一起了
    )

rois_per_image = cfg.TRAIN.BATCH_SIZE / num_images #cfg.TRAIN.BATCH_SIZE为128
  #cfg.TRAIN.FG_FRACTION为0.25，即在一次分类训练中前景框只能有32个
  fg_rois_per_image = np.round(cfg.TRAIN.FG_FRACTION * rois_per_image)

#  选择需要分类的框，并求他们的类别和坐标的gt，以及计算边框损失loss时所需要的bbox_inside_weights
labels, rois, roi_scores, bbox_targets, bbox_inside_weights = _sample_rois(
    all_rois, all_scores, gt_boxes, fg_rois_per_image, rois_per_image, _num_classes))
```
**train_model() -> construnct_graph() -> create_architecture() -> _build_network() -> _region_proposal() -> proposal_target_layer() ->_sample_rois() **
```python
"""
    生成前景样本和背景样本
    计算rois与gt_bboxes之间的overlap矩阵，对于每一个roi，最大的overlap的gt_bbox的标签即为该roi的类别标签，
    并根据TRAIN.FG_THRESH和TRAIN.BG_THRESH_HI/LO 选择前景roi和背景roi
"""
 # 计算所有roi和gt框之间的重合度IoU
overlaps = bbox_overlaps(
    np.ascontiguousarray(all_rois[:, 1:5], dtype=np.float),
    np.ascontiguousarray(gt_boxes[:, :4], dtype=np.float))
    
# 对于每个roi找到对应的gt_box坐标shape[len(all_rois),]
gt_assignment = overlaps.argmax(axis=1) 
# 对于每个roi，找到与gt_box重复度最高的overlap shape[len(all_rois),]
max_overlaps = overlaps.max(axis=1)
# 对于每个roi，找到对应归属的类别
labels = gt_boxes[gt_assignment, 4]

# 找到属于前景的rois（就是与gt_box覆盖率超过0.5的）
  fg_inds = np.where(max_overlaps >= cfg.TRAIN.FG_THRESH)[0]
# 找到属于背景的rois(就是与gt_box覆盖介于0和0.5之间的)
  bg_inds = np.where((max_overlaps < cfg.TRAIN.BG_THRESH_HI) &
                     (max_overlaps >= cfg.TRAIN.BG_THRESH_LO))[0]
                     
# 得到最终保留框的类别gt值，以及坐标变换gt值
  bbox_target_data = _compute_targets(
    rois[:, 1:5], gt_boxes[gt_assignment[keep_inds], :4], labels)
    
# 得到最终计算loss时使用的gt的边框回归值和bbox_inside_weights
  bbox_targets, bbox_inside_weights = \
    _get_bbox_regression_labels(bbox_target_data, num_classes)
```
**train_model() -> construnct_graph() -> create_architecture() -> _build_network() -> _region_proposal() -> proposal_target_layer() ->_sample_rois() -> _get_bbox_regression_labels()**
```python
# 求得最终计算loss时使用的groundtruth边框回归值和bbox_inside_weights
clss = bbox_target_data[:, 0] # 先得到每个用来训练的每个roi的类别
  bbox_targets = np.zeros((clss.size, 4 * num_classes), dtype=np.float32) 
  #用全0初始化一下边框回归的ground truth值。针对每个roi，对每个类别都置4个坐标回归值
  bbox_inside_weights = np.zeros(bbox_targets.shape, dtype=np.float32) #用全0初始化一下bbox_inside_weights
  inds = np.where(clss > 0)[0] # 找到属于前景的rois
  for ind in inds: # 针对每一个前景roi
    cls = clss[ind] # 找到其所属类别
    start = int(4 * cls) #找到从属的类别对应的坐标回归值的起始位置
    end = start + 4 #找到从属的类别对应的坐标回归值的结束位置
    bbox_targets[ind, start:end] = bbox_target_data[ind, 1:] #在对应类的坐标回归上置相应的值
    bbox_inside_weights[ind, start:end] = cfg.TRAIN.BBOX_INSIDE_WEIGHTS
  return bbox_targets, bbox_inside_weights
```
#### ROI Pooling网络
FC layer需要固定尺寸的输入。在最早的R-CNN算法中，将输入的图像直接resize成相同的尺寸。而Faster R-CNN对输入图像的尺寸没有要求，经过Proposal layer和 Proposal target layer之后，会得到许多不同尺寸的RoI。Faster R-CNN采用RoI Pooling层（原理参考SPPNet 论文），将不同尺寸ROI对应的特征图采样为相同尺寸，然后输入后续的FC层。这版代码中没有实现RoI pooling layer， 而是把RoI对应的特征图resize成相同尺寸后，再进行max pooling。

>具体原理参见 弄懂Faster RCNN.md 一文
**train_model() -> construnct_graph() -> create_architecture() ->_build_network() ->_crop_pool_layer()**
```python
def _crop_pool_layer(self, bottom, rois, name):
    with tf.variable_scope(name) as scope:
      batch_ids = tf.squeeze(tf.slice(rois, [0, 0], [-1, 1], name="batch_id"), [1])
      # Get the normalized coordinates of bounding boxes
      # 得到归一化的坐标，（相对于原图的尺寸进行归一化）
      bottom_shape = tf.shape(bottom)
      height = (tf.to_float(bottom_shape[1]) - 1.) * np.float32(self._feat_stride[0])
      width = (tf.to_float(bottom_shape[2]) - 1.) * np.float32(self._feat_stride[0])
      x1 = tf.slice(rois, [0, 1], [-1, 1], name="x1") / width
      y1 = tf.slice(rois, [0, 2], [-1, 1], name="y1") / height
      x2 = tf.slice(rois, [0, 3], [-1, 1], name="x2") / width
      y2 = tf.slice(rois, [0, 4], [-1, 1], name="y2") / height
      # Won't be back-propagated to rois anyway, but to save time
      bboxes = tf.stop_gradient(tf.concat([y1, x1, y2, x2], axis=1))
      pre_pool_size = cfg.POOLING_SIZE * 2
      # 裁剪特征图，并resize成相同的尺寸，#进行ROI pool，之所以需要归一化框的坐标是因为tf接口的要求
      # 变成14 x 14的大小
      crops = tf.image.crop_and_resize(bottom, bboxes, tf.to_int32(batch_ids), [pre_pool_size, pre_pool_size], name="crops")
    #用2×2的滑动窗口进行最大池化操作，输出的尺度是7×7
    return slim.max_pool2d(crops, [2, 2], padding='SAME')

    """
      处理方法：直接将ROI区域对应的特征分割出来，并按照某一尺度(14*14)进行线性双插值resize，再使用max_pool2d池化将特征变为7*7
    """
```
#### VGG16网络的fc层
**train_model() -> construnct_graph() -> create_architecture() ->_head_to_tail()**
```python
fc7 = self._head_to_tail(pool5, is_training)

def _head_to_tail(self, pool5, is_training, reuse=None):
    with tf.variable_scope(self._scope, self._scope, reuse=reuse):
      pool5_flat = slim.flatten(pool5, scope='flatten') # 将输入扁平化，但保留batch
      fc6 = slim.fully_connected(pool5_flat, 4096, scope='fc6')
      if is_training:
        fc6 = slim.dropout(fc6, keep_prob=0.5, is_training=True, 
                            scope='dropout6')
      fc7 = slim.fully_connected(fc6, 4096, scope='fc7')
      if is_training:
        fc7 = slim.dropout(fc7, keep_prob=0.5, is_training=True, 
                            scope='dropout7')
    return fc7
```
#### 分类回归网络
**train_model() -> construnct_graph() -> create_architecture() ->_region_classification()**
```python
def _region_classification(self, fc7, is_training, initializer, initializer_bbox):
    cls_score = slim.fully_connected(fc7, self._num_classes, 
                                       weights_initializer=initializer,
                                       trainable=is_training,
                                       activation_fn=None, scope='cls_score')
    cls_prob = self._softmax_layer(cls_score, "cls_prob")
    cls_pred = tf.argmax(cls_score, axis=1, name="cls_pred")
    bbox_pred = slim.fully_connected(fc7, self._num_classes * 4, 
                                     weights_initializer=initializer_bbox,
                                     trainable=is_training,
                                     activation_fn=None, scope='bbox_pred')

    self._predictions["cls_score"] = cls_score
    self._predictions["cls_pred"] = cls_pred
    self._predictions["cls_prob"] = cls_prob
    self._predictions["bbox_pred"] = bbox_pred
```
_build_network()结束！
### 总结
- 其主要的工作
    - 构建VGG16网络
    - 生成anchors
    - 生成RPN网络
        - 1、预测anchor的类别，(属于前景和背景)及其位置
        ```python
        self._predictions["rpn_cls_score"] = rpn_cls_score
        self._predictions["rpn_cls_score_reshape"] = rpn_cls_score_reshape
        self._predictions["rpn_cls_prob"] = rpn_cls_prob
        self._predictions["rpn_cls_pred"] = rpn_cls_pred
        self._predictions["rpn_bbox_pred"] = rpn_bbox_pred
        self._predictions["rois"] = rois
        ```
        - 2、生成RPN网络的标签信息(anchor target layer)
        ```python
        self._anchor_targets['rpn_labels'] = rpn_labels
        self._anchor_targets['rpn_bbox_targets'] = rpn_bbox_targets
        self._anchor_targets['rpn_bbox_inside_weights'] = rpn_bbox_inside_weights
        self._anchor_targets['rpn_bbox_outside_weights'] = rpn_bbox_outside_weights
        ```
        - 3、生成训练分类和回归网络的ROI(proposal layer)以及对应的标签信息(proposal target layer)
        ```python
        self._proposal_targets['rois'] = rois
        self._proposal_targets['labels'] = tf.to_int32(labels, name="to_int32")
        self._proposal_targets['bbox_targets'] = bbox_targets
        self._proposal_targets['bbox_inside_weights'] = bbox_inside_weights
        self._proposal_targets['bbox_outside_weights'] = bbox_outside_weights
        ```
    - 生成ROI Pooling
    - 生成分类回归网络
        - 生成回归预测后的边框和score
        ```python
        self._predictions["cls_score"] = cls_score
        self._predictions["cls_pred"] = cls_pred
        self._predictions["cls_prob"] = cls_prob
        self._predictions["bbox_pred"] = bbox_pred 
        ```
### 定义损失loss
**train_model() -> construnct_graph() -> create_architecture() ->_add_losses()**
```python
"""
# 定义损失loss，包括两部分：
    1、RPN的分类和回归
    2、RCNN的分类和回归
    
    分类：sparse_softmax_cross_entropy_with_logits   回归：_smooth_l1_loss
"""
# RPN 分类
rpn_cross_entropy = tf.reduce_mean(
        tf.nn.sparse_softmax_cross_entropy_with_logits(logits=rpn_cls_score, labels=rpn_label))
# RPN 回归
rpn_loss_box = self._smooth_l1_loss(rpn_bbox_pred, rpn_bbox_targets, rpn_bbox_inside_weights,
                                          rpn_bbox_outside_weights, sigma=sigma_rpn, dim=[1, 2, 3])
# RCNN分类
cross_entropy = tf.reduce_mean(tf.nn.sparse_softmax_cross_entropy_with_logits(logits=cls_score, labels=label))
# RCNN 回归
loss_box = self._smooth_l1_loss(bbox_pred, bbox_targets, bbox_inside_weights, bbox_outside_weights)

# 包括四项损失，RPN的分类和回归，RCNN的分类和回归，分类都是用的softmax_cross_entropy,回归损失都是用的smooth_L1_loss
loss = cross_entropy + loss_box + rpn_cross_entropy + rpn_loss_box
```
- 到此，create_architecture结束，回到construct_graph()函数。

### 定义优化器
用的是MomentumOptimizer
```python
self.optimizer = tf.train.MomentumOptimizer(lr, cfg.TRAIN.MOMENTUM)

# compute_gradients()+apply_gradients() = minimize()
gvs = self.optimizer.compute_gradients(loss)
train_op = self.optimizer.apply_gradients(final_gvs)
```

- 到此，准备已经完毕，接下来开始数据进行训练。
### 梯度下降法计算梯度，获取一个batch
```java
blobs = self.data_layer.forward() # 用梯度下降法训练，一次获取一个batch
                \\
                 \\
                 layer/_get_next_minibatch()
                                \\
                                 \\
                                 db_inds = self._get_next_minibatch_inds() #得到一组新batch的index
                                                \\
                                                 \\
                                                  self.shuffle_roidb_inds() # 打乱索引顺序
                                 get_minibatch(minibatch_db, self._num_classes) #获取索引对应的roidb
                                                \\
                                                 \\
                                                  _get_image_blob()
                                                        \\
                                                         \\
                                                         im,im_scale=prep_im_for_blob() #将原始图像尺寸PxQ变成网络的输入MxN的大小
                                                         blob=im_list_to_blob() #将图像的list变为网络的输入，转成blob格式
```


