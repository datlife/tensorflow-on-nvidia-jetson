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
2. [Install protobuf](#2-install-protobuf)
3. [Install grpc-java](#3-install-grpc-java)
4. [Install bazel](#4-install-bazel)
6. [Install Tensorflow](#5-install-tensorflow)
  * Download and configure tensorflow
  * Set CUDA v7.0 and CuDNN 4 as a compiler
  * Modify few library files to work on ARM device

#### References
* [Tensorflow on Rasberry Pi 3](https://www.neotitans.net/install-tensorflow-on-odroid-c2.html)
* [Tensorflow on ODROID-C2](https://www.neotitans.net/install-tensorflow-on-odroid-c2.html)
* [TensorFlow on Jetson TK1](http://cudamusing.blogspot.com/2016/06/tensorflow-08-on-jetson-tk1.html)


### 1. Install Dependencies
---------------------------

```shell
# For protobuf
sudo apt-get install autoconf automake libtool curl make g++ unzip maven

# For bazel
sudo will-be-updated

# For TensorFlow (I assumed you are using python 2 on Jetson TK1. If you are using python 3, I am not sure if the rest will work)
# For Python 2.7
sudo apt-get install python-pip python-numpy swig python-dev
sudo pip install wheel

# Optimization Flags
sudo apt-get install gcc-4.8 g++-4.8
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 100
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.8 100
```

2. Install protobuf
-------------------

In order to install grpc-java, we need new version of protobuf
 * Download and checkout 3.1.0 version
```shell
cd $HOME
git clone https://github.com/google/protobuf.git
cd protobuf
git checkout v3.1.0
```
 * Build and Install `protobuf 3.1.0`
```shell
./autogen.sh
LDFLAGS=-static ./configure --prefix=$(pwd)/../
sed -i -e 's/LDFLAGS = -static/LDFLAGS = -all-static/' ./src/Makefile
make -j 4
sudo make install
```

However, we need older version protobuf v3.0.0-beta-4 to build bazel. Notice that I did not run `sudo make install`, so system still uses v3.1.0. 
 * Build `protobuf 3.0.0-beta2`
```shell
git checkout v3.0.0-beta-2
./autogen.sh
LDFLAGS=-static ./configure --prefix=$(pwd)/../
sed -i -e 's/LDFLAGS = -static/LDFLAGS = -all-static/' ./src/Makefile
make -j 4
```
* Check your protobuf installation. It should say `libprotoc 3.1.0`
```shell
protoc --version
```

3. Install grpc-java
--------------------

* Download and checkout v0.15.0
```shell
cd ~ && git clone https://github.com/grpc/grpc-java.git | cd grpc-java
git checkout v0.15.0
```
* Edit file `grpc-java\compiler\build.gradle` so it could build on Linux ARM32
```shell
cd ..\compiler
vim build.gradle
```
* Add the following lines to the file
```shell
# you can show line numbers in vim by :set number

# Around line 49.
    ...
    gcc(Gcc) {
         target("linux_arm-v7"){
           cppCompiler.executable="/usr/bin/gcc"
    }
    ...
# Around line 65. below x86_64 add    

    'linux_arm-v7' {
        architecture "armv7l"
        operatingSystem "linux"
    }

# Edit the linkers around lines 100. Removes and replace with
    linker.args "-static", "-lprotoc", "-lprotobuf", "-static-libgcc",
                "-static-libstdc++", "-lpthread", "-s"

```
* Install 'grpc-java' using `protobuf` of the system
```shell
CXXFLAGS="-I$(pwd)/../include" LDFLAGS="-L$(pwd)/../lib" ./gradlew java_pluginExecutable -Pprotoc=$(pwd)/../bin/protoc
```
* You should see something like this
```shell
*** Building codegen requires Protobuf version 3.0.0-beta-3
*** Please refer to https://github.com/grpc/grpc-java/blob/master/COMPILING.md#how-to-build-code-generation-plugin
:grpc-compiler:compileJava_pluginExecutableJava_pluginCpp UP-TO-DATE
:grpc-compiler:linkJava_pluginExecutable UP-TO-DATE
:grpc-compiler:java_pluginExecutable UP-TO-DATE

BUILD SUCCESSFUL

Total time: 16.264 secs

This build could be faster, please consider using the Gradle Daemon: https://docs.gradle.org/2.13/userguide/gradle_daemon.html

```

4. Install bazel
----------------

* Download `bazel 0.4.3`
```shell
cd ~
wget https://github.com/bazelbuild/bazel/releases/download/0.4.3/bazel-0.4.3-dist.zip
unzip -d bazel bazel-0.4.3-dist.zip
```
* Configure file `./compile.sh`:
```shell
vim ./compile.sh
# Around line 30. Change ${VERBOSE:=no} to ${VERBOSE:=yes}
${VERBOSE:=yes}
```
* Configure file `tools/cpp/cc_configure.bzl` . The issue has been discuess on [here](https://github.com/bazelbuild/bazel/issues/1264)
```shell
# Add an additional if statement to  _get_cpu_value()
result = repository_ctx.execute(["uname", "-m"])
machine_cpu = result.stdout.strip()
if machine_cpu in ["arm", "armv7l", "aarch64"]:
	return "arm"
return "k8" if machine in ["amd64", "x86_64", "x64"] else "piii"
```
 * Build `bazel` with `protoc v3.0.0-beta-2` and `grpc-java`
```shell
PROTOC=../protobuf/src/protoc
GRPC_JAVA_PLUGIN=../grpc-java/compiler/build/exe/java_plugin/protoc-gen-grpc-java
sudo ./compile.sh
```
 * Result output
```shell
 You can skip this first step by providing a path to the bazel binary as second argument:
INFO:    ./compile.sh compile /path/to/bazel
 Building Bazel from scratch../usr/lib/jvm/java-8-oracle/bin/javac ....
 ......
 ......# Wait around 5-10 mins
 

Target //src:bazel up-to-date: bazel-bin/src/bazel
INFO: Elapsed time: 593.067s, Critical Path: 573.93s
WARNING: /tmp/bazel_O9OVPiDR/out/external/bazel_tools/WORKSPACE:1: Workspace name in /tmp/bazel_O9OVPiDR/out/external/bazel_tools/WORKSPACE (@io_bazel) does not match the name given in the repository's definition (@bazel_tools); this will cause a build error in future versions.

Build successful! Binary is here: /home/ubuntu/bazel/output/bazel

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

### B. Set up `CUDA 7.0` and `cuDNN v4` as compiler for TensorFlow
-------------------------------------------------------------------

The reason is that TF supports only CUDA 7.0 and up. Although we cannot use CUDA 7.0 on TK1, we can still install to use it as a compiler.

 * Download and install CUDA 7.0
```shell
wget http://developer.download.nvidia.com/embedded/L4T/r24_Release_v1.0/CUDA/cuda-repo-l4t-7-0-local_7.0-76_armhf.deb
sudo dpkg -i cuda-repo-l4t-7-0-local_7.0-76_armhf.deb 
sudo apt-get update && sudo apt-get install cuda-toolkit-7-0
```

* Remove symlink `CUDA 7.0` created and link to `CUDA 6.5` instead
```shell
sudo rm /usr/local/cuda
sudo ln -s /usr/local/cuda-6.5/ /usr/local/cuda
sudo ln -s /usr/local/cuda-6.5/lib/libcudnn.so /usr/local/cuda-6.5/lib/libcudnn.so.2

```
* Download `cuDNN 7.0` to use during compilation
```shell
# Download  cuDNN 4 and decompress
tar -xvf cudnn-7.0-linux-ARMv7-v4.0-prod.tgz 
cd cudnn/cuda/

# Will use this later in next step
```

5. Install TensorFlow
---------------------

### A. Create swap disk
 ----------------------
For safety of overloading the disk, people suggest to use external swap disk (USB) to install TensorFlow.
* Find the USB path
```shell
lsblk

NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    1  29.5G  0 disk 		 # <------ This is my usb
`-sda1        8:1    1  29.5G  0 part [SWAP] 
mmcblk0rpmb 179:16   0     4M  0 disk 
mmcblk0     179:0    0  14.7G  0 disk 
```
* Unmount the disk and create swap disk
```shell
sudo umount /dev/sda
sudo mkswap /dev/sda
sudo swapon /dev/sda 
```

### B. Download and Configure TensorFlow
----------------------------------------
* Download TensorFlow and checkout v0.12.1
```shell
git clone --recurse-submodules https://github.com/tensorflow/tensorflow
cd tensorflow
git checkout v0.12.1
```

* Replace all `lib-64` with `lib` and configure TF before installation.
```shell
grep -Rl "lib64"| xargs sed -i 's/lib64/lib/g'
```


* Replace all `lib-64` with `lib` and configure TF before installation.
```shell
./configure
CUDA support will be enabled for TensorFlow
Please specify which gcc should be used by nvcc as the host compiler. [Default is /usr/bin/gcc]: 
Please specify the CUDA SDK version you want to use, e.g. 7.0. [Leave empty to use system default]: 
Please specify the location where CUDA 7.0 toolkit is installed. Refer to README.md for more details. [Default is /usr/local/cuda]: 
Please specify the Cudnn version you want to use. [Leave empty to use system default]: 
Please specify the location where cuDNN  library is installed. Refer to README.md for more details. [Default is /usr/local/cuda-6.5]: 
Please specify a list of comma-separated Cuda compute capabilities you want to build with.
You can find the compute capability of your device at: https://developer.nvidia.com/cuda-gpus.
Please note that each additional compute capability significantly increases your build time and binary size.
[Default is: "3.5,5.2"]: 3.2

INFO: Starting clean (this may take a while). Consider using --expunge_async if the clean takes more than several minutes.
......................
INFO: All external dependencies fetched successfully.
Configuration finished
``` 

### C. Modify few libraries
---------------------------

* Edit `tensorflow/core/platform/platform.h` as following because it possibly causes error like [this](https://github.com/tensorflow/tensorflow/issues/3469)
```shell
#define IS_MOBILE_PLATFORM   <----- DELETE THIS LINE
```

* Edit the following files to avoid TensoFlow crashed ([here](http://cudamusing.blogspot.com/2016/06/tensorflow-08-on-jetson-tk1.html)).

.* First one : `tensorflow/core/kernels/conv_ops_gpu_2.cu.cc`
```shell
#ifndef __arm__
template struct functor::InflatePadAndShuffle<GPUDevice, Eigen::half, 4, Eigen::DenseIndex>;
template struct functor::InflatePadAndShuffle<GPUDevice, float, 4,Eigen::DenseIndex>;
#endif
```

* Second one : `tensorflow/core/kernels/conv_ops_gpu_3.cu.cc`
```shell
#ifndef __arm__
template struct functor::ShuffleAndReverse<GPUDevice, float, 4, Eigen::DenseIndex>;
template struct functor::ShuffleAndReverse<GPUDevice, Eigen::half, 4, Eigen::DenseIndex>;
#endif
```

* Third one : `tensorflow/stream_executor/cuda/cuda_gpu_executor.cc`
```shell
static int TryToReadNumaNode(const string &pci_bus_id, int device_ordinal) {
#ifdef __arm__
  LOG(INFO) << "ARMV7 does not support NUMA - returning NUMA node zero";
  return 0;
#elif defined(__APPLE__)
```

* Fourth one: `tensorflow/core/common_runtime/gpu/process_state.cc`
```shell
if (kCudaHostMemoryUseBFC) {
      allocator =
#ifdef __arm__
          new BFCAllocator(new CUDAHostAllocator(se), 1LL << 31, true /*allow_growth*/, "cuda_host_bfc" /*name*/);
#else
          new BFCAllocator(new CUDAHostAllocator(se), 1LL << 36 /*64GB max*/, true /*allow_growth*/, "cuda_host_bfc" /*name*/);
#endif
```

* Fifth file :`tensorflow/core/kernels/cwise_op_gpu_select.cu.cc`
 
````shell
 # Around line 43
#if !defined(EIGEN_HAS_INDEX_LIST)
	//Eigen::array<int, 2> broadcast_dims{{ 1, all_but_batch }};
	Eigen::array<int, 2> broadcast_dims;
	broadcast_dims[0] = 1;
	broadcast_dims[1] = all_but_batch;
	// Eigen::Tensor<int, 2>::Dimensions reshape_dims{{ batch, 1 }};
	Eigen::Tensor<int, 2>::Dimensions reshape_dims;
	reshape_dims[0] = batch;
	reshape_dims[1] = 1;
 #else
    Eigen::IndexList<Eigen::type2index<1>, int> broadcast_dims;
```
 
* Sixth file : `tensorflow/core/kernels/sparse_tensor_dense_matmul_op_gpu.cu.cc`
````shell
#if !defined(EIGEN_HAS_INDEX_LIST)
	// Eigen::array<int, 1> reduce_on_rows{{ 0 }};
	Eigen::Tensor<int, 2>::Dimensions matrix_1_by_nnz;
	matrix_1_by_nnz[0] = 1;  
	matrix_1_by_nnz[1] = nnz;
	// Eigen::array<int, 2> n_by_1{{ n, 1 }};
	Eigen::array<int, 2> n_by_1;
	n_by_1[0] = n; 
	n_by_1[1] = 1;
	// Eigen::array<int, 1> reduce_on_rows{{ 0 }};
	Eigen::array<int, 1> reduce_on_rows;
	reduce_on_rows[0]= 0;
#else
Eigen::IndexList<Eigen::type2index<1>, int> matrix_1_by_nnz;
```

* 1st Installation. As having mentioned by [cudamusing](), we will wait for first fail so that we could configure the `Macro.h` file.
```shell
bazel build -c opt --jobs 1 --local_resources 1800,2.0,1.0 --verbose_failures --config=cuda //tensorflow/tools/pip_package:build_pip_package
bazel build -c opt --jobs 1 --local_resources 1800,2.0,1.0 --verbose_failures --config=cuda //tensorflow/cc:tutorials_example_trainer
```

* 2nd Installation. When it failed, edit `Marco.h` file in ` `~/.cache/bazel/_bazel_ubuntu/ad1e09741bb4109fbc70ef8216b59ee2/external/eigen_archive/Eigen/src/Core/util/Macros.h` . Notice my hash number `ad1...` could be different than yours.
```shell
vim ~/.cache/bazel/_bazel_ubuntu/ad1e09741bb4109fbc70ef8216b59ee2/external/eigen_archive/Eigen/src/Core/util/Macros.h
# Around line 400

// Does the compiler support variadic templates?
#ifndef EIGEN_HAS_VARIADIC_TEMPLATES
#if EIGEN_MAX_CPP_VER>=11 && (__cplusplus > 199711L || EIGEN_COMP_MSVC >= 1900) \
# --> remove this//  && (!defined(__NVCC__) || !EIGEN_ARCH_ARM_OR_ARM64 || (defined __CUDACC_VER__ && __CUDACC_VER__ >= 80000) )
// ^^ Disable the use of variadic templates when compiling with versions of nvcc older than 8.0 on ARM devices:
 //    this prevents nvcc from crashing when compiling Eigen on Tegra X1
#define EIGEN_HAS_VARIADIC_TEMPLATES 1


# After finished, save the file and restart the build
bazel build -c opt --jobs 1 --local_resources 1800,2.0,1.0 --verbose_failures --config=cuda //tensorflow/tools/pip_package:build_pip_package
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

* Congratulations! You have succesfully built TensorFlow from source on NVIDA Jetson TK1


#### Known Issues during compilation

1. Ran out of memory. Try to update `--local-resoures` where n1,n2,n3 is memroy,cpu_thread,i/o input

```shell
C++ compilation of rule '//tensorflow/core/kernels:svd_op' failed: gcc failed: error executing command -
```
