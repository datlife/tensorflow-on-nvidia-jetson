Build Tensorflow from source
============================

### NVIDIA Jetson TK1 (Linux 32-bit armv7l)

#### Basic Requirements
* Time. ``This process might take you hours.``
* An external memory as swap disk. An USB is okay.
* NVIDIA Jetson TK1 + Ubuntu 14.04.1

#### Summary
1. Install dependencies
2. Install protobuf
3. Install grpc-java
4. Install bazel
5. Install Tensorflow



-----------------------
1. Install Dependencies
-----------------------

protobuf
```shell
sudo apt-get install autoconf automake libtool curl make g++ unzip maven
```
Bazel
```shell
sudo will-be-updated
```
TensorFlow (I assumed you are using python 2 on Jetson TK1. If you are using python 3, I am not sure if the rest will work)
```shell
# For Python 2.7
sudo apt-get install python-pip python-numpy swig python-dev
sudo pip install wheel
```
Optimization Flags
```shell
sudo apt-get install gcc-4.8 g++-4.8
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 100
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.8 100
```

-----
## 2. Install protobuf

In order to install grpc-java, we need new version of protobuf
```shell
cd $HOME
git clone https://github.com/google/protobuf.git
cd protobuf
git checkout v3.1.0
./autogen.sh
LDFLAGS=-static ./configure --prefix=$(pwd)/../
sed -i -e 's/LDFLAGS = -static/LDFLAGS = -all-static/' ./src/Makefile
make -j 4
sudo make install
```
However, we need older version protobuf v3.0.0-beta-4 to build bazel. `sudo make install` is not run so system still uses v3.1.0
```shell
git checkout v3.0.0-beta-2
./autogen.sh
LDFLAGS=-static ./configure --prefix=$(pwd)/../
sed -i -e 's/LDFLAGS = -static/LDFLAGS = -all-static/' ./src/Makefile
make -j 4
```
Check your protobuf installation. It should say libprotoc 3.1.0
```shell
protoc --version
```
