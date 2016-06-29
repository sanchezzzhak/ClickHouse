# How to build ClickHouse

Build should work on Linux Ubuntu 12.04, 14.04 or newer.
With appropriate changes, build should work on any other Linux distribution.
Build is not intended to work on Mac OS X.
Only x86_64 with SSE 4.2 is supported. Support for AArch64 is experimental.

To test for SSE 4.2, do
```
grep -q sse4_2 /proc/cpuinfo && echo "SSE 4.2 supported" || echo "SSE 4.2 not supported"
```

## Install Git and CMake

```
sudo apt-get install git cmake
```

## Detect number of threads
```
export THREADS=$(grep -c ^processor /proc/cpuinfo)
```

## Install GCC 5

(GCC 6 is fine too)
There are several ways to do it.

### 1. If you run on Ubuntu 15.10 or newer, just do
```
sudo apt-get install g++-5
```

### 2. Install from PPA package.

```
sudo apt-get install software-properties-common
sudo apt-add-repository ppa:ubuntu-toolchain-r/test
sudo apt-get update
sudo apt-get install gcc-5 g++-5
```

### 3. Install GCC 5 from sources.

Example:
```
# Download gcc from https://gcc.gnu.org/mirrors.html
wget ftp://ftp.fu-berlin.de/unix/languages/gcc/releases/gcc-5.3.0/gcc-5.3.0.tar.bz2
tar xf gcc-5.3.0.tar.bz2
cd gcc-5.3.0
./contrib/download_prerequisites
cd ..
mkdir gcc-build
cd gcc-build
../gcc-5.3.0/configure --enable-languages=c,c++
make -j $THREADS
sudo make install
hash gcc g++
gcc --version
sudo ln -s /usr/local/bin/gcc /usr/local/bin/gcc-5
sudo ln -s /usr/local/bin/g++ /usr/local/bin/g++-5
sudo ln -s /usr/local/bin/gcc /usr/local/bin/cc
sudo ln -s /usr/local/bin/g++ /usr/local/bin/c++
# /usr/local/bin/ should be in $PATH
```

Note that these ways of installation differs.
When installing from PPA, by default, "old C++ ABI" is used,
 and when installing from sources, "new C++ ABI" is used.
When using different C++ ABI, you need to recompile all C++ libraries,
 otherwise libraries will not link.
ClickHouse works with both old and new C++ ABI,
 but production releases is built with old C++ ABI.

## Use GCC 5 for builds

```
export CC=gcc-5
export CXX=g++-5
```

## Install required libraries from packages

```
sudo apt-get install libicu-dev libglib2.0-dev libreadline-dev libmysqlclient-dev libssl-dev unixodbc-dev libzookeeper-mt-dev
```

## Install recent version of boost

Version 1.57 or newer will be Ok.

There are several ways to do it.

### 1. Install from sources.

```
wget http://downloads.sourceforge.net/project/boost/boost/1.60.0/boost_1_60_0.tar.bz2
tar xf boost_1_60_0.tar.bz2
cd boost_1_60_0
./bootstrap.sh
./b2 --toolset=gcc-5 -j $THREADS
sudo ./b2 install --toolset=gcc-5 -j $THREADS
cd ..
```

### 2. Install from package.

On Ubuntu 15.10 or newer.
```
sudo apt-get install libboost-dev libboost-thread-dev libboost-program-options-dev libboost-system-dev libboost-regex-dev libboost-filesystem-dev
```

## Install mongoclient (optional)

This library is needed only for 'external dictionaries' with MongoDB source.
This is rarely used but enabled by default.

If you don't need it, you could set variable and skip installation step:
```
export DISABLE_MONGODB=1
```

Otherwise:
```
sudo apt-get install scons
git clone -b legacy https://github.com/mongodb/mongo-cxx-driver.git
cd mongo-cxx-driver
sudo scons --c++11 --release --cc=$CC --cxx=$CXX --ssl=0 --disable-warnings-as-errors -j $THREADS --prefix=/usr/local install
cd ..
```

# Checkout ClickHouse sources

```
git clone git@github.com:yandex/ClickHouse.git
# or: git clone https://github.com/yandex/ClickHouse.git

cd ClickHouse
```

Note that master branch is not stable.
For stable version, switch to some release branch.

# Build ClickHouse

There are two variants of build.
## 1. Build release package.

### Install prerequisites to build debian packages.
```
sudo apt-get install devscripts dupload fakeroot debhelper
```

### Install recent version of clang.

Clang is embedded into ClickHouse package and used at runtime. Minimal version is 3.8.0. It is optional.

There are two variants:
#### 1. Build clang from sources.
```
cd ..
sudo apt-get install subversion
mkdir llvm
cd llvm
svn co http://llvm.org/svn/llvm-project/llvm/tags/RELEASE_380/final llvm
cd llvm/tools
svn co http://llvm.org/svn/llvm-project/cfe/tags/RELEASE_380/final clang
cd ..
cd projects/
svn co http://llvm.org/svn/llvm-project/compiler-rt/tags/RELEASE_380/final compiler-rt
cd ../..
mkdir build
cd build/
cmake -D CMAKE_BUILD_TYPE:STRING=Release ../llvm
make -j $THREADS
sudo make install
hash clang
```

#### 2. Install from packages.

On Ubuntu 16.04 or newer:
```
sudo apt-get install clang
```

You may also build ClickHouse with clang for development purposes.
For production releases, GCC is used.

### Run release script.
```
rm -f ../clickhouse*.deb
./release --standalone
```

You will find built packages in parent directory.
```
ls -l ../clickhouse*.deb
```

Note that usage of debian packages is not required.
ClickHouse has no runtime dependencies except libc,
 so it could work on almost any Linux.

### Installing just built packages on development server.
```
sudo dpkg -i ../clickhouse*.deb
sudo service clickhouse-server start
```

## 2. Build to work with code.
```
mkdir build
cd build
cmake ..
make -j $THREADS
cd ..
```
