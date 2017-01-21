Build Tensorflow from source
============================

### NVIDIA Jetson TK1 (Linux 32-bit armv7l)

#### Basic Requirements
* NVIDIA Jetson TK1 + Ubuntu 14.04.1
* CUDA 6.5 + CuDNN installed
* An external memory as swap disk. An USB is okay.
* Time. This process might take you hours.


#### Summary
1. Install dependencies
2. Install protobuf
3. Install grpc-java
4. Install bazel
5. Install Tensorflow

#### References
* [Tensorflow on Rasberry Pi 3](https://www.neotitans.net/install-tensorflow-on-odroid-c2.html)
* [Tensorflow on ODROID-C2](https://www.neotitans.net/install-tensorflow-on-odroid-c2.html)
* [TensorFlow on Jetson TK1](http://cudamusing.blogspot.com/2016/06/tensorflow-08-on-jetson-tk1.html)

-----------------------
1. Install Dependencies
-----------------------

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

3. Install bazel
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

```
