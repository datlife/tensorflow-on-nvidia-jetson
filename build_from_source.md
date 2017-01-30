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
5. [Build Tensorflow](#5-build-tensorflow)
  * Create Swap Disk
  * Set CUDA v7.0 and CuDNN 4 as a compiler
  * Build TensorFlow
6. [Install Tensorflow](#6-install-tensorflow)

#### References
* [Tensorflow on Rasberry Pi 3](https://www.neotitans.net/install-tensorflow-on-odroid-c2.html)
* [Tensorflow on ODROID-C2](https://www.neotitans.net/install-tensorflow-on-odroid-c2.html)
* [TensorFlow on Jetson TK1](http://cudamusing.blogspot.com/2016/06/tensorflow-08-on-jetson-tk1.html)


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
 * If download link gmock is failed
```shell
vim WORKSPACE
vim autogen.sh

# Update this link 
http://pkgs.fedoraproject.org/repo/pkgs/gmock/gmock-1.7.0.zip/073b984d8798ea1594f5e44d85b20d66/gmock-1.7.0.zip

# https://github.com/grpc/grpc/issues/7952

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

* Download `cuDNN 7.0` to use during compilation
```shell
# Download  cuDNN 4 and decompress
tar -xvf cudnn-7.0-linux-ARMv7-v4.0-prod.tgz 
cd cudnn/cuda/

# Copy to cuda folder
sudo cp ./cudnn/cuda/include/cudnn.h /usr/local/cuda/include/
```

5. Build TensorFlow
---------------------

#### A. Create swap disk
 -----------------------
For safety of overloading the memory, people suggest to use external swap disk (USB) to install TensorFlow.

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

#### B. Download and Configure TensorFlow
-----------------------------------------
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
grep -Rl "lib64"| xargs sed -i 's/lib64/lib/g'
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


#### Known Issues during compilation

1. Error with NVIDIA DRIVER. I got some weird error such as `could not find cuDevicePrimaryCtxSetFlags in libcuda` . The issue I suspect due to incompatible driver.
```
sudo apt-get install libcuda1-367

```
2. Ran out of memory. Try to update `--local-resoures` where n1,n2,n3 is memroy,cpu_thread,i/o input

```shell
C++ compilation of rule '//tensorflow/core/kernels:svd_op' failed: gcc failed: error executing command -
```
