Build Tensorflow from source
============================

### NVIDIA Jetson TK1 (Linux 32-bit armv7l)

#### Basic Requirements
* Time. This process might take you hours.
* An external memory as swap disk.
* NVIDIA Jetson TK1 + Ubuntu 14.04.1

#### Summary
1. Install dependencies
2. Install protobuf
3. Install grpc-java
4. Install bazel
5. Install Tensorflow


1. Install Dependencies
-----------------------

* protobuf
```shell
sudo apt-get install autoconf automake libtool curl make g++ unzip maven
```
* Bazel
```shell
sudo will-be-updated
```
* TensorFlow (I assumed you are using python 2 on Jetson TK1. If you are using python 3, I am not sure if the rest will work)
```shell
# For Python 2.7
sudo apt-get install python-pip python-numpy swig python-dev
sudo pip install wheel
```
* Optimization Flags
```shell
sudo apt-get install gcc-4.8 g++-4.8
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 100
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.8 100
```

-----
## 2. Install protobuf
