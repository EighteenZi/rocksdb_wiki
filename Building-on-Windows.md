# Building on Windows

This is a simple step-by-step explanation of how I was able to build RocksDB (or RocksJava) and all of the 3rd-party libraries on Microsoft Windows 10. The Windows build system was already in place, however it took some trial-and-error for me to be able to build the 3rd-party libraries and incorporate them into the build.

## Pre-requisites
1. Microsoft Visual Studio 2015 (Community)
2. [CMake](https://cmake.org/)
3. [Git](https://git-scm.com/downloads) - I used the Windows Git Bash.
4. [Mercurial](https://www.mercurial-scm.org/wiki/Download) - I used the 64bit MSI installer
5. wget

## Steps

Create a directory somewhere on your machine that will be used a container for both the RocksDB source code and that of its 3rd-party dependencies. On my machine I used `C:\Users\aretter\code`, from hereon in I will just refer to it as `%CODE_HOME%`; which can be set as an environment variable, i.e. `SET CODE_HOME=C:\Users\aretter\code`.

All of the following is executed from the "**Developer Command Prompt for VS2015**":

### Build GFlags
```
cd %CODE_HOME%
wget https://github.com/gflags/gflags/archive/v2.2.0.zip
unzip v2.2.0.zip
cd gflags-2.2.0
mkdir target
cd target
cmake -G "Visual Studio 14 Win64" ..
```

Open the project in Visual Studio, create a new x64 Platform by copying the Win32 platform and selecting x64 CPU. Close Visual Studio.

```
msbuild gflags.sln /p:Configuration=Debug /p:Platform=x64
msbuild gflags.sln /p:Configuration=Release /p:Platform=x64
```

The resultant static library can be found in `%CODE_HOME%\gflags-2.2.0\target\lib\Debug\gflags_static.lib` or `%CODE_HOME%\gflags-2.2.0\target\lib\Debug\gflags_static.lib`.


### Build Snappy
```
cd %CODE_HOME%
hg clone https://bitbucket.org/robertvazan/snappy-visual-cpp
cd snappy-visual-cpp
hg diff --reverse -r 44:45 > exports.patch
hg import exports.patch
devenv snappy.sln /upgrade
msbuild snappy.sln /p:Configuration=Debug /p:Platform=x64
msbuild snappy.sln /p:Configuration=Release /p:Platform=x64
```

The resultant static library can be found in `%CODE_HOME%\snappy-visual-cpp\x64\Debug\snappy64.lib` or `%CODE_HOME%\snappy-visual-cpp\x64\Release\snappy64.lib`.


### Build LZ4
```
cd %CODE_HOME%
wget https://github.com/lz4/lz4/archive/v1.7.5.zip
unzip v1.7.5.zip
cd lz4-1.7.5
cd visual\VS2010
devenv lz4.sln /upgrade
msbuild lz4.sln /p:Configuration=Debug /p:Platform=x64
msbuild lz4.sln /p:Configuration=Release /p:Platform=x64
```

The resultant static library can be found in `%CODE_HOME%\lz4-1.7.5\visual\VS2010\bin\x64_Debug\liblz4_static.lib` or `%CODE_HOME%\lz4-1.7.5\visual\VS2010\bin\x64_Release\liblz4_static.lib`.


### Build ZLib
```
cd %CODE_HOME%
wget http://zlib.net/zlib1211.zip
unzip zlib1211.zip
cd zlib-1.2.11\contrib\vstudio\vc14
"C:\Program Files (x86)\Microsoft Visual Studio 14.0\VC\bin\amd64_x86\vcvarsamd64_x86.bat"
msbuild zlibvc.sln /p:Configuration=Debug /p:Platform=x64
msbuild zlibvc.sln /p:Configuration=Release /p:Platform=x64
x64\ZlibDllRelease\zlibwapi.lib
```

The resultant static library can be found in `%CODE_HOME%\zlib-1.2.11\contrib\vstudio\vc14\x64\ZlibDllDebug\zlibwapi.lib` or `%CODE_HOME%\zlib-1.2.11\contrib\vstudio\vc14\x64\ZlibDllRelease\zlibwapi.lib`.


### Build RocksDB
```
cd %CODE_HOME%
git clone https://github.com/facebook/rocksdb.git
cd rocksdb
```

Edit the file `%CODE_HOME%\rocksdb\thirdparty.inc` to have these changes:

```
set(GFLAGS_HOME $ENV{THIRDPARTY_HOME}/gflags-2.2.0)
set(GFLAGS_INCLUDE ${GFLAGS_HOME}/target/include)
set(GFLAGS_LIB_DEBUG ${GFLAGS_HOME}/target/lib/Debug/gflags_static.lib)
set(GFLAGS_LIB_RELEASE ${GFLAGS_HOME}/target/lib/Release/gflags_static.lib)

set(SNAPPY_HOME $ENV{THIRDPARTY_HOME}/snappy-visual-cpp)
set(SNAPPY_INCLUDE ${SNAPPY_HOME})
set(SNAPPY_LIB_DEBUG ${SNAPPY_HOME}/x64/Debug/snappy64.lib)
set(SNAPPY_LIB_RELEASE ${SNAPPY_HOME}/x64/Release/snappy64.lib)

set(LZ4_HOME $ENV{THIRDPARTY_HOME}/lz4-1.7.5)
set(LZ4_INCLUDE ${LZ4_HOME}/lib)
set(LZ4_LIB_DEBUG ${LZ4_HOME}/visual/VS2010/bin/x64_Debug/liblz4_static.lib)
set(LZ4_LIB_RELEASE ${LZ4_HOME}/visual/VS2010/bin/x64_Release/liblz4_static.lib)

set(ZLIB_HOME $ENV{THIRDPARTY_HOME}/zlib-1.2.11)
set(ZLIB_INCLUDE ${ZLIB_HOME})
set(ZLIB_LIB_DEBUG ${ZLIB_HOME}/contrib/vstudio/vc14/x64/ZlibDllDebug/zlibwapi.lib)
set(ZLIB_LIB_RELEASE ${ZLIB_HOME}/contrib/vstudio/vc14/x64/ZlibDllRelease/zlibwapi.lib)
```

And then finally to compile RocksDB:
```
mkdir build
cd build
set THIRDPARTY_HOME=%CODE_HOME%
cmake -G "Visual Studio 14 Win64" -DJNI=1 -DGFLAGS=1 -DSNAPPY=1 -DLZ4=1 -DZLIB=1 -DXPRESS=1 ..
msbuild rocksdb.sln /p:Configuration=Release
```
