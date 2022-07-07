#  More ConvNets in the 2020s: Scaling up Kernels Beyond 51 × 51 using Sparsity

Official PyTorch implementation of **SLaK**, from the following paper: 

More ConvNets in the 2020s: Scaling up Kernels Beyond 51 x 51 using Sparsity. 

[Shiwei Liu](https://shiweiliuiiiiiii.github.io/), [Tianlong Chen](https://tianlong-chen.github.io/about/), [Xiaohan Chen](http://www.xiaohanchen.com/), [Xuxi Chen](https://xxchen.site/), Qiao Xiao, Boqian Wu, [Mykola Pechenizkiy](https://www.win.tue.nl/~mpechen/), [Decebal Mocanu](https://people.utwente.nl/d.c.mocanu), [Zhangyang Wang](https://vita-group.github.io/)\
Eindhoven University of Technology, University of Texas at Austin, University of Twente

--- 
<p align="center">
<img src="https://github.com/Shiweiliuiiiiiii/SLaK/blob/main/SLaK.png" width="500" height="300">
</p>

We propose **SLaK**, a pure ConvNet model that for the first time is able to scale the convolutional kernels up to 51x51. 

## Catalog
- [x] ImageNet-1K Training Code   
- [x] ImageNet-1K Fine-tuning Code  
- [x] Downstream Transfer (Detection, Segmentation) Code


<!-- ✅ ⬜️  -->

## Results and Pre-trained Models
### ImageNet-1K trained models

| name | resolution | kernel size |acc@1 | #params | FLOPs | model |
|:---:|:---:|:---:|:---:| :---:|:---:|:---:|
| SLaK-T | 224x224 | 51x51 |82.5 | 30M | 5.0G | to be upload |
| SLaK-S | 224x224 | 51x51 | 83.7 | 55M | 9.8G |  to be upload |
| SLaK-B | 224x224 | 51x51 | 84.0 | 95M | 17.1G |  to be upload |

## Installation

The code is tested used CUDA 11.3.1, cudnn 8.2.0, PyTorch 1.10.0 with A100 GPUs.

### Dependency Setup
Create an new conda virtual environment
```
conda create -n slak python=3.8 -y
conda activate slak
```

Install [Pytorch](https://pytorch.org/)>=1.10.0. For example:
```
conda install pytorch==1.11.0 torchvision==0.12.0 torchaudio==0.11.0 cudatoolkit=11.3 -c pytorch
```

Clone this repo and install required packages:
```
git clone https://github.com/Shiweiliuiiiiiii/SLaK.git
pip install timm tensorboardX six
```

To enable training SLaK, we first need to install the efficient large-kernel convolution with PyTorch provided in https://github.com/MegEngine/cutlass/tree/master/examples/19_large_depthwise_conv2d_torch_extension by the following steps:

1. Clone ```cutlass``` (https://github.com/MegEngine/cutlass), enter the directory.
2. ```cd examples/19_large_depthwise_conv2d_torch_extension```
3. ```./setup.py install --user```. If you get errors, (1) check your ```CUDA_HOME```; (2) you might need to change the source code a bit to make tensors contiguous see [here](https://github.com/Shiweiliuiiiiiii/SLaK/blob/3f8b1c46eee34da440afae507df13bc6307c3b2c/depthwise_conv2d_implicit_gemm.py#L25) for example. 
4. A quick check: ```python depthwise_conv2d_implicit_gemm.py```
5. Add ```WHERE_YOU_CLONED_CUTLASS/examples/19_large_depthwise_conv2d_torch_extension``` into your ```PYTHONPATH``` so that you can ```from depthwise_conv2d_implicit_gemm import DepthWiseConv2dImplicitGEMM``` anywhere. Then you may use ```DepthWiseConv2dImplicitGEMM``` as a replacement of ```nn.Conv2d```.
6. ```export LARGE_KERNEL_CONV_IMPL=WHERE_YOU_CLONED_CUTLASS/examples/19_large_depthwise_conv2d_torch_extension``` so that RepLKNet will use the efficient implementation. Or you may simply modify the related code (```get_conv2d```) in ```replknet.py```.

## Training code

We provide ImageNet-1K training, and ImageNet-1K fine-tuning commands here.

### ImageNet-1K SLaK-T on a single machine
```
python -m torch.distributed.launch --nproc_per_node=4 main.py  \
--Decom True --sparse --width_factor 1.3 -u 100 --sparsity 0.4 --sparse_init snip  --prune_rate 0.3 \
--epochs 300 --model SLaK_tiny --drop_path 0.1 --batch_size 128 \
--lr 4e-3 --update_freq 8 --model_ema true --model_ema_eval true \
--data_path /path/to/imagenet-1k --num_workers 40 \
--kernel-size 51 49 47 13 5 --output_dir /path/to/save_results
```

- You can add `--use_amp true` to train in PyTorch's Automatic Mixed Precision (AMP).
- Use `--resume /path_or_url/to/checkpoint.pth` to resume training from a previous checkpoint; use `--auto_resume true` to auto-resume from latest checkpoint in the specified output folder.
- `--batch_size`: batch size per GPU; `--update_freq`: gradient accumulation steps.
- The effective batch size = `--nodes` * `--ngpus` * `--batch_size` * `--update_freq`. In the example above, the effective batch size is `4*8*128*1 = 4096`. You can adjust these four arguments together to keep the effective batch size at 4096 and avoid OOM issues, based on the model size, number of nodes and GPU memory.
- `--sparse`: enable sparse model; `--sparsity`: model sparsity; `--width_factor`: model width; `-u`: adaptation frequency; `--prune_rate`: adaptation rate.

### ImageNet-1K SLaK-S on a single machine
```
python -m torch.distributed.launch --nproc_per_node=4 main.py  \
--Decom True --sparse --width_factor 1.3 -u 100 --sparsity 0.4 --sparse_init snip  --prune_rate 0.3 \
--epochs 300 --model SLaK_small --drop_path 0.1 --batch_size 128 \
--lr 4e-3 --update_freq 8 --model_ema true --model_ema_eval true \
--data_path /path/to/imagenet-1k --num_workers 40 \
--kernel-size 51 49 47 13 5 --output_dir /path/to/save_results
```
