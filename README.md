Install TensorFlow on NVIDA Jetson TK1
========================================
### TL;DR
Download set-up file: 
```shell
wget https://github.com/dat-ai/tensorfow-nvidia-jetson/
```
Install file with pip
```shell
sudo pip install
```shell
Test TenforFlow on NVIDIA Jetson TK1
```shell
python
import tensorflow as tf
```

-----
### Introduction

After spending countless hours to make it working properly, I decided to share my set-up file to make others' life easier. NVIDA Jetson and Tensorflow is not easy to install due to the hardware constraint of the Jetson. 

TensorFlow mostly support CUDA 7 and up. However, Jetson TK1's CUDA is 6.5. 

#### Contents
* Install using pip (easy)
* Build from source (hard and not fun) - If you need to build newer TensorFLow.
