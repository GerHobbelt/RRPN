# RRPN++: Guidance Towards More Accurate Scene Text Detection
Report can be viewed at: https://arxiv.org/abs/2009.13118

# Hightlights

- 89.5 F-measure in single scale in ICDAR 2015 benchmark (https://rrc.cvc.uab.es/?ch=4&com=evaluation&view=method_info&task=1&m=78081).
- 92.0 F-measure in single scale in ICDAR 2013 and testing speed can reach 13.3 fps with 640px (tested in single GPU of GTX 1080Ti).
- Adopting RRoI Align in Detectron2.
- Text Spotter with Transformer (training and testing).
- Support for higher pytorch verison >=1.7.
- Bug fixed for rbox cannot detect angle over 45.

## Environments
- Ubuntu 16.04
- Cuda 10 or 9
- python >=3.5
- **pytorch >= 1.7 (Higher version supported)** 
- Other packages like numpy, cv2.


![alt text](demo/visualization.png "Results from IC15 testing set")


## Installation

Check [INSTALL.md](INSTALL.md) for installation instructions.

## Configuring your dataset
- Your dataset path can be set in `$RRPN_ROOT/maskrcnn_benchmark/config/paths_catalog.py`. We implemented interface for {IC13, IC15, IC17mlt, LSVT, ArT} for common use(Start from line 96):
```bash
...
 "RRPN_train": {  # including IC13 and IC15
            'dataset_list':{
                # 'IC13': 'Your dataset path',
                ...
            },
            "split": 'train'
        },
...
```
- Add your dataset in detector?
You need to form a dict array as follows:
```bash
im_info = {
    'gt_classes': your class_id array,
    'max_classes': your class_id array,
    'image': path to access one image,
    'boxes': rotate box in {cx, cy, w, h, θ},
    'flipped': Not supported, just False, 
    'gt_overlaps': overlaps fill with 1 (gt with gt),
    'seg_areas': H * W for an rbox,
    'height': height of an image,
    'width': width of an image,
    'max_overlaps': overlaps fill with 1 (gt with gt),
    'rotated': just True
}
```
Examples can be seen in `$RRPN_ROOT/maskrcnn_benchmark/data/rotation_series.py`
Your data API should be add to the variable `DATASET`:
```bash
DATASET = {
    'IC13':get_ICDAR2013,
    'IC15':get_ICDAR2015_RRC_PICK_TRAIN,
    'IC17mlt':get_ICDAR2017_mlt,
    ...
    'Your Dataset Name': 'Your Dataset API'
}
```

- Add your dataset in spotter?
You need to form a dict array as follows:
```bash
im_info = {
    'gt_classes': your class_id array,
    'max_classes': your class_id array,
    'image': path to access one image,
    'boxes': rotate box in {cx, cy, w, h, θ},
    'flipped': Not supported, just False, 
    'gt_overlaps': overlaps fill with 1 (gt with gt),
    'seg_areas': H * W for an rbox,
    'height': height of an image,
    'width': width of an image,
    'gt_words': words of each box,
    'max_overlaps': overlaps fill with 1 (gt with gt),
    'rotated': just True
}
```
Examples can be seen in `$RRPN_ROOT/maskrcnn_benchmark/data/rrpn_e2e_series.py`
Your data API should be add to the variable `DATASET`:
```bash
DATASET = {
    'IC13':get_ICDAR2013,
    'IC15':get_ICDAR2015_RRC_PICK_TRAIN,
    'IC17mlt':get_ICDAR2017_mlt,
    ...
    'Your Dataset Name': 'Your Dataset API'
}
```

## Training 
```bash
# create your data cache directory
mkdir data_cache
```

Train a detector of RRPN++

```bash
# 使用faster-rcnn训练检测器
python tools/train_net.py --config-file=configs/arpn/e2e_rrpn_R_50_C4_1x_train_AFPN_RT_LERB.yaml

# 使用mask-rcnn训练检测器
python tools/train_net.py --config-file=configs/Mask_RRPN/e2e_rrpn_R_50_C4_1x_LSVT_train_MASK_RFPN.yaml
```
# 训练检测器注意：
1.模型训练预处理部分都在这个文件中maskrcnn_benchmark/data/transforms/build.py
   arpn/rrpn/rpn三种结构定义了各自的预处理类
   其中，RandomCrop、RandomRotation可能导致rpn_loss出现Nan，如果出现Nan,注释这两个预处理类
   debug发现会导致训练模型时输入到模型的target为Nan，最终导致rpn loss计算时出现loss，该问题可能由问题2一起导致loss出现Nan  
2.模型训练rrpn结构代码存在bug，位置在maskrcnn_benchmark/modeling/rrpn/loss.py中的64行：   
     labels_per_image[~anchors_per_image.get_field("visibility")] = -1  
     该行代码存在bug，导致positive classmaskrcnn_benchmark/modeling/rrpn/loss.py（前景类）类别数为0，最终导致梯度爆炸，loss为Nan，如果出现，注释掉改行代码  
3. 模型训练arpn结构代码存在bug，位置在maskrcnn_benchmark/modeling/arpn/loss.py中的88行：  
     right_bound = box_areas < size_range[1]  
     该行对box的面积进行过滤，阈值太小会将图片的所有box全部过滤掉，导致loss出现Nan  
     解决方法：修改该文件的213行，调大面积阈值  
     size_range = [0, self.size_stack[-3] ** 2]  
4. 学习率过大loss也会出现Nan，一般最大为0.0001

Train a spotter（端到端） (Used in RRPN++ report and we strongly recommand to use) of RRPN++

```bash
# In your root of RRPN
python tools/train_net.py --config-file=configs/arpn_E2E/e2e_rrpn_R_50_C4_1x_train_AFPN_RT_LERB_Spotter.yaml
```

- Multi-GPU phase is not testing yet, be careful to use GPU more than 1.

## Testing
- Using  `$RRPN_ROOT/demo/ICDAR19_eval_script.py` or `$RRPN_ROOT/demo/rrpn_e2e_infer.py`(Strongly recommanded) to test images you want. The demo will generate a text for your detected coodinates.
- Showing the detected image by ture the variable `vis` to True.

- By adding the following setting into your configure yaml to test the datasets, or you can re-implement the file to test your images.
- One of the configure file we recommand is `$RRPN_ROOT/configs/arpn_E2E/e2e_rrpn_R_50_C4_1x_test_AFPN_RT_LERB_Spotter.yaml`
- Choose the dataset you want to evaluate on.

```bash
TEST:
  DATASET_NAME: "IC15" # Choice can be "IC15", "LSVT" and so on
  MODE: "DET" # DET for detection evaluation or E2E for recognition results in the spotter
```


## More Results 

## Final 
- Enjoy it with all the codes.
- Citing us if you find it work in your projects.

```
@article{ma2020rrpn++,
  title={RRPN++: Guidance Towards More Accurate Scene Text Detection},
  author={Ma, Jianqi},
  journal={arXiv preprint arXiv:2009.13118},
  year={2020}
}
```

## Special Thanks
- My family for FIRM SUPPORT of devices and power supply.
- Jingye Chen's (https://github.com/JinGyeSetBirdsFree) support of the code and report.
