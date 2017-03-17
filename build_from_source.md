Build Tensorflow from source
============================

### NVIDIA Jetson TK1 (Linux 32-bit armv7l)

#### Basic Requirements
* NVIDIA Jetson TK1 + Ubuntu 14.04.1
* CUDA 6.5 + CuDNN installed
* An external memory as swap disk. An USB is okay.
* Time. This process might take you hours.


#### Summary
1. [Install dependencies](#1-install-dependencies)
2. [Install bazel 0.4.4](#2-install-bazel) -- 
3. [Build Tensorflow](#3-build-tensorflow)
  * Create Swap Disk
  * Set CUDA v7.0 and CuDNN 4 as a compiler
  * Build TensorFlow
6. [Install Tensorflow](#6-install-tensorflow)

#### References
* [Tensorflow on Rasberry Pi 3](https://www.neotitans.net/install-tensorflow-on-odroid-c2.html)
* [Tensorflow on ODROID-C2](https://www.neotitans.net/install-tensorflow-on-odroid-c2.html)
* [TensorFlow on Jetson TK1](http://cudamusing.blogspot.com/2016/06/tensorflow-08-on-jetson-tk1.html)

#### UPDATE LOGS

03/16/2017: 
Bazel 4.4 fully supports ARM32-bit. Therefore, we do not have to build protobuf and gRPCJava from starch



### 1. Install Dependencies
---------------------------
```shell
* Install Java JDK 8.0
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java8-installer

# For protobuf & bazel. We will also need `maven`, but I rather install it after this step.
sudo apt-get install git zip unzip autoconf automake libtool curl zlib1g-dev  

# For TensorFlow (I assumed you are using python 2 on Jetson TK1. If you are using python 3, I am not sure if the rest will work)
# For Python 2.7
sudo apt-get install python-pip python-numpy swig python-dev
sudo pip install wheel
```

### 2. Install Bazel
--------------------

* Download `bazel 0.4.4`
```shell
wget https://github.com/bazelbuild/bazel/releases/download/0.4.4/bazel-0.4.4-dist.zip
mkdir bazel-0.4.4
unzip bazel-0.4.4-dist.zip -d bazel-0.4.4
cd bazel-0.4.4
```

* Create swap disk
Since Jetson TK1 only has 2GB, it is safe practice to install extra swap
```shell
# Find the USB path
lsblk

NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    1  29.5G  0 disk 		 # <------ This is my usb
`-sda1        8:1    1  29.5G  0 part [SWAP] 
...
```

* Unmount the disk and create swap disk
```shell
sudo umount /dev/sda
sudo mkswap /dev/sda
sudo swapon /dev/sda 
```

 * Copy `bazel` to `/usr/local/bin`.
```shell
 sudo cp output/bazel /usr/local/bin/bazel
``` 

 * Verify that bazel is working
```shell
 $ bazel

Extracting Bazel installation...
.....................
                                               [bazel release 0.4.3- (@non-git)]
Usage: bazel <command> <options> ...

Available commands:
  analyze-profile     Analyzes build profile data.
  build               Builds the specified targets.
...
```

**Optional : **
If you get  `OutOfResource` error durring installtion. Try to manually configure maximum heap size as following
* Configure file `scripts/bootstrap/compile.sh` :
```shell
# Limit heap size
vim scripts/bootstrap/compile.sh
# Around line 137
# Add this part `-J-Xmx2g`   <---Set maximum Heap Size to 2GB
run "${JAVAC}" -classpath "${classpath}" -sourcepath "${sourcepath}" \
      -d "${output}/classes" -source "$JAVA_VERSION" -target "$JAVA_VERSION" \
      -encoding UTF-8 "@${paramfile}" -J-Xmx2g
```


### B. Set up `CUDA 7.0` and `cuDNN v3` as compiler for TensorFlow
-------------------------------------------------------------------
The reason is that TF supports only CUDA 7.0 and up. Although we cannot use CUDA 7.0 on TK1, we can still install to use it as a compiler.

 * Download and install CUDA 7.0
```shell
wget http://developer.download.nvidia.com/embedded/L4T/r24_Release_v1.0/CUDA/cuda-repo-l4t-7-0-local_7.0-76_armhf.deb
sudo dpkg -i cuda-repo-l4t-7-0-local_7.0-76_armhf.deb 
sudo apt-get update && sudo apt-get install cuda-toolkit-7-0
```
Note:
If you run into error like the following, run `sudo apt-get clean` and try again
```
dpkg-deb (subprocess): decompressing archive member: internal gzip read error: '<fd:4>: incorrect data check'
```

* Download `cuDNN 3.0` to use during compilation
```shell
# Download  cuDNN 3.0 and decompress

tar -xvf cudnn-7.0-linux-ARMv7-v3.0-prod.tgz 
cd cudnn/cuda/

# Copy to cuda folder
sudo cp ./cudnn/cuda/include/cudnn.h /usr/local/cuda/include/
```

5. Build TensorFlow
---------------------

* Download TensorFlow and checkout v0.12.1
```shell
git clone --recurse-submodules https://github.com/tensorflow/tensorflow
cd tensorflow
git checkout v0.12.1
```

* Apply the patch for TensorFlow v0.12.1. Please note that if you are building different version. The patch will not work. Basically, we eddited some files in the tensorflow source so it allows us to compile on jetson TK1.
```shell
patch -p1 < ../tensorflow_0.12.1_jetsontk1.patch
```

* Replace all `lib-64` with `lib` and configure TF before installation.
```shell
sudo grep -Rl 'lib64' | sudo xargs sed -i 's/lib64/lib/g'
```

* Configure TF before installation (~./.bashrc and cuDNN is set up correctly).
```shell
./configure
...
Please note that each additional compute capability significantly increases your build time and binary size.
[Default is: "3.5,5.2"]: **`3.2`**
INFO: Starting clean (this may take a while). Consider using --expunge_async if the clean takes more than several minutes.
......................
INFO: All external dependencies fetched successfully.
Configuration finished
``` 

#### C. Install TensorFlow
--------------------------

* First Installation: wait for failure to edit `Macro.h` file (mentioned by [cudamusing]() )
```shell
bazel build -c opt --jobs 1 --local_resources 1800,0.5,1.0 --verbose_failures --config=cuda //tensorflow/tools/pip_package:build_pip_package
```

* When it failed, edit `Marco.h` file in ` `~/.cache/bazel/_bazel_ubuntu/ad1e09741bb4109fbc70ef8216b59ee2/external/eigen_archive/Eigen/src/Core/util/Macros.h` . Notice my hash dir could be different than yours.
```shell
#ifndef EIGEN_HAS_VARIADIC_TEMPLATES
#if EIGEN_MAX_CPP_VER>=11 && (__cplusplus > 199711L || EIGEN_COMP_MSVC >= 1900) \
// -->  && (!defined(__NVCC__) || !EIGEN_ARCH_ARM_OR_ARM64 || (defined __CUDACC_VER__ && __CUDACC_VER__ >= 80000) )
// ^^ Disable the use of variadic templates when compiling with versions of nvcc older than 8.0 on ARM devices:
//    this prevents nvcc from crashing when compiling Eigen on Tegra X1
#define EIGEN_HAS_VARIADIC_TEMPLATES 1
```

* Second Installation
```shell
bazel build -c opt --jobs 1 --local_resources 1800,0.5,1.0 --verbose_failures --config=cuda //tensorflow/tools/pip_package:build_pip_package
```

* If it is successfully built, you should see something like this
```shell
Target //tensorflow/tools/pip_package:build_pip_package up-to-date:
  bazel-bin/tensorflow/tools/pip_package/build_pip_package
INFO: Elapsed time: 7153.308s, Critical Path: 333.40s
```

6. Install TensorFlow
---------------------

* Build .whl package
```shell
bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg
```

* Install .whl package
```shell
sudo pip install /tmp/tensorflow_pkg/tensorflow-0.12.1-cp27-none-linux_armv7l.whl
```


* Remove symlink `CUDA 7.0` in order to run Tensorflow with `CUDA 6.5`
```shell
sudo rm /usr/local/cuda
sudo ln -s /usr/local/cuda-6.5/ /usr/local/cuda

# Update symlinks lib*.7.0 to local one

# Watch out the driver cuda-driver-dev-6-5
```

* Congratulations! You have succesfully built TensorFlow from source on NVIDA Jetson TK1

* Test Tensorflow
```shell
# https://www.tensorflow.org/how_tos/using_gpu/
python
import tensorflow as tf

a = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[2, 3], name='a')
b = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[3, 2], name='b')
c = tf.matmul(a, b)
# Creates a session with log_device_placement set to True.
sess = tf.Session(config=tf.ConfigProto(log_device_placement=True))
# Runs the op.
print sess.run(c)

# Expected Output
I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:872] ARMV7 does not support NUMA -returning NUMA node zero
I tensorflow/core/common_runtime/gpu/gpu_init.cc:102] Found device 0 with properties: 
name: GK20A
major: 3 minor: 2 memoryClockRate (GHz) 0.852
pciBusID 0000:00:00.0
Total memory: 1.85GiB
Free memory: 199.25MiB
I tensorflow/core/common_runtime/gpu/gpu_init.cc:126] DMA: 0 
I tensorflow/core/common_runtime/gpu/gpu_init.cc:136] 0:   Y 
I tensorflow/core/common_runtime/gpu/gpu_device.cc:755] Creating TensorFlow device (/gpu:0) -> (device: 0, name: GK20A, pci bus id: 0000:00:00.0)
Device mapping:
/job:localhost/replica:0/task:0/gpu:0 -> device: 0, name: GK20A, pci bus id: 0000:00:00.0
I tensorflow/core/common_runtime/direct_session.cc:149] Device mapping:
/job:localhost/replica:0/task:0/gpu:0 -> device: 0, name: GK20A, pci bus id: 0000:00:00.0

>>> print sess.run(c)
b: /job:localhost/replica:0/task:0/gpu:0
I tensorflow/core/common_runtime/simple_placer.cc:388] b: /job:localhost/replica:0/task:0/gpu:0
a: /job:localhost/replica:0/task:0/gpu:0
I tensorflow/core/common_runtime/simple_placer.cc:388] a: /job:localhost/replica:0/task:0/gpu:0
MatMul: /job:localhost/replica:0/task:0/gpu:0
I tensorflow/core/common_runtime/simple_placer.cc:388] MatMul: /job:localhost/replica:0/task:0/gpu:0
[[ 22.  28.]
 [ 49.  64.]]
>>> 

```


#### Known Issues during compilation

2. Ran out of memory. Try to update `--local-resoures` where n1,n2,n3 is memroy,cpu_thread,i/o input

```shell
C++ compilation of rule '//tensorflow/core/kernels:svd_op' failed: gcc failed: error executing command -
```
