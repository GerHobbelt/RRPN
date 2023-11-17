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

# Use `faster-rcnn` to train the detector
python tools/train_net.py --config-file=configs/arpn/e2e_rrpn_R_50_C4_1x_train_AFPN_RT_LERB.yaml

# Use `mask-rcnn` to train the detector
python tools/train_net.py --config-file=configs/Mask_RRPN/e2e_rrpn_R_50_C4_1x_LSVT_train_MASK_RFPN.yaml
```

# Training detector Notes:

1. The model training preprocessing part is in this file: `maskrcnn_benchmark/data/transforms/build.py`
   The three structures of arpn/rrpn/rpn define their own preprocessing classes.
   Among them, `RandomCrop and RandomRotation` may cause `rpn_loss` to appear as `NaN`. 
   If `NaN` appears, comment out these two preprocessing classes.
   `debug` found that it will cause the `target` input to the model to be `NaN` when training the model, 
   which will eventually cause loss to appear when calculating rpn loss. 
   This problem may be caused by problem 2 together and cause the `loss` to appear as `NaN`.
   
2. There is a bug in the model training `rrpn` structure code, located at line 64 in `maskrcnn_benchmark/modeling/rrpn/loss.py`:

       labels_per_image[~anchors_per_image.get_field("visibility")] = -1

   There is a bug in this line of code, which causes the number of categories 
   in positive `classmaskrcnn_benchmark/modeling/rrpn/loss.py` (foreground class) to be 0, 
   which eventually leads to gradient explosion and `loss` showing up as `NaN`. 
   If it occurs, comment out and change the line of code.     
   
3. There may be a bug in the model training `arpn` structure code, located at line 88 in `maskrcnn_benchmark/modeling/arpn/loss.py`:

       right_bound = box_areas < size_range[1]
       
   This line filters the area of ​​the box. If the threshold is too small, all boxes in the image will be filtered out, 
   which may cause the loss to appear as `NaN` (to be confirmed).
   
   Solution: Modify line 213 of the file and increase the area threshold:
   
       size_range = [0, self.size_stack[-3] ** 2]
          
4. Loss will also appear as `NaN` if the learning rate is too large, generally the maximum is 0.0001

5. If the data set crosses the boundary, it will definitely lead to a `NaN` loss.
   The Tianchi `icpr` data set was used during training. After training for a period of time, 
   `NaN` will appear. Even if `lr` (learning rate) is very small, `NaN` will appear, 
   which is difficult to troubleshoot.
   
Train a spotter (end-to-end) (Used in `RRPN++` report and we strongly recommand to use) of RRPN++

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
