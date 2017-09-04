Install TensorFlow on NVIDA Jetson TK1
========================================
### TL;DR
Install dependencies for TensorFlow:
```shell
sudo apt-get update
sudo apt-get install python-pip python-dev
```

Clone this repo: 
```shell
git clone https://github.com/dat-ai/tensorfow-nvidia-jetson/
cd tensorfow-nvidia-jetson
```

Install file with pip
```shell
sudo pip install tensorflow-0.8.0-py2-none-any.whl
```

Test TenforFlow on NVIDIA Jetson TK1
```shell
python
import tensorflow as tf
```
-----
