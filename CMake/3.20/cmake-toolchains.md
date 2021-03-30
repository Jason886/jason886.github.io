## cmake-toolchains

[TOC]

### 介绍

CMake通过使用工具链程序实现编译、链接、归档、以及其他构建任务。CMake根据当前已启用的程序语言自动检测可用的工具链程序。在通常的构建中（指host build），CMake基于系统信息来自动检测工具链。在交叉编译的情况下（cross-compile），可以通过指定编译器或工具路径的方式来指定工具链。



### 程序语言

通过`project()`指令可以设定程序语言。通过`project()`指令设定程序语言时，将会设定一些语言相关的内建变量，比如：`CMAKE_CXX_COMPILER`，`CMAKE_CXX_COMPILER_ID`等等。如果在顶级的CMakeLists文件中没有写`project()`指令，CMake将会隐式的生成一个，此时默认启用的程序语言是C和CXX：

```cmake
project(C_Only C)
```

也可以使用`project()`指令来设定不启用任何语言：

```cmake
project(MyProject NONE)
```

在`project()`指令之后，可以再通过`enable_language()`指令来启用语言：

```cmake
enable_language(CXX)
```

当启用一个语言的时候，CMake将会寻找该语言的编译器，并且检测一些信息如编译器的厂商和版本、编译目标的架构和位宽、对应的使用程序的位置等等。

`ENABLED_LANGUAGES`全局变量含有当前已启用的全部程序语言。



### 变量和属性

有一些变量关联到工具链的语言组件。`CMAKE_<LANG>_COMPILER`是`<LANG>`语言编译器的绝对路径。`CMAKE_<LANG>_COMPILER_ID`是CMake用来标识编译器的。`CMAKE_<LANG>_COMPILER_VERSION`是编译器的版本。

`CMAKE_<LANG>_FLAGS`变量和 configuration-specific equivalents 包含将在编译`<LANG>`语言时被添加到编译命令中的标志。

由于链接器是通过编译器来调用的，CMake需要一个方式来决定哪些编译器可用来调用链接器。This is calculated by the [`LANGUAGE`](https://cmake.org/cmake/help/v3.20/prop_sf/LANGUAGE.html#prop_sf:LANGUAGE) of source files in the target, and in the case of static libraries, the language of the dependent libraries. The choice CMake makes may be overridden with the [`LINKER_LANGUAGE`](https://cmake.org/cmake/help/v3.20/prop_tgt/LINKER_LANGUAGE.html#prop_tgt:LINKER_LANGUAGE) target property.



### 工具链特性

CMake提供了`try_compile()`指令和一些包装宏比如`CheckCXXSourceCompiles`、`CheckCXXSymbolExists`和`CheckIncludeFile`来测试各种工具链特性是否可用。这些API测试工具链然后把结果缓存，以便下次运行CMake时不要再次执行这些测试。

在CMake中对某些编译器特性内建了处理，不需要compile-test。例如：`POSITION_INDEPENDENT_CODE`用来指定一个目标应该编译为位置无关的代码，如果编译器支持该特性的话。`<LANG>_VISIBILITY_PRESET`和`VISIBILITY_INLINE_HIDDEN`目标属性添加标志 for hidden visibility，如果编译器支持的话。



### 交叉编译

如果在调用cmake的时候在命令行参数中指定了 `-DCMAKE_TOOLCHAIN_FILE=path/to/file`，这个文件将会提前加载，用来设置编译器。如果CMake进行交叉编译，`CMAKE_CROSSCOMPILING`变量将会被设置为true。

注意，避免在一个toolchain文件中使用`CMAKE_SOURCE_DIR`或`CMAKE_BINARY_DIR`。The toolchain file is used in contexts where these variables have different values when used in different places (e.g. as part of a call to [`try_compile()`](https://cmake.org/cmake/help/v3.20/command/try_compile.html#command:try_compile)). In most cases, where there is a need to evaluate paths inside a toolchain file, the more appropriate variable to use would be [`CMAKE_CURRENT_LIST_DIR`](https://cmake.org/cmake/help/v3.20/variable/CMAKE_CURRENT_LIST_DIR.html#variable:CMAKE_CURRENT_LIST_DIR), since it always has an unambiguous, predictable value.



#### 交叉编译 for Linux

一个典型的Linux交叉编译toolchain文件的内容：

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_SYSROOT /home/devel/rasp-pi-rootfs)
set(CMAKE_STAGING_PREFIX /home/devel/stage)

set(tools /home/devel/gcc-4.7-linaro-rpi-gnueabihf)
set(CMAKE_C_COMPILER ${tools}/bin/arm-linux-gnueabihf-gcc)
set(CMAKE_CXX_COMPILER ${tools}/bin/arm-linux-gnueabihf-g++)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

`CMAKE_SYSTEM_NAME`是目标平台的标识。

`CMAKE_SYSTEM_PROCESS`是目标架构的标识。

`CMAKE_SYSROOT`是可选的，当需要设定sysroot时可以设定之。

`CMAKE_STAGIN_PREFIX`也是可选的。可以为其设定一个host上的路径，用来安装到其中。`CMAKE_INSTALL_PREFIX`也是安装路径，即使在交叉编译时也生效。

`CMAKE_<LANG>_COMPILER`可以被设为绝对路径，或者编译器的名称（将会在标准位置搜索）。对于不支持<u>不带标志或脚本</u>进行链接的工具链，需要设定`CMAKE_TRY_COMPILE_TARGET_TYPE`变量为`STATIC_LIBRARY`来告诉CMake不要在检测阶段尝试链接可执行文件。

CMake `find_*` commands will look in the sysroot, and the [`CMAKE_FIND_ROOT_PATH`](https://cmake.org/cmake/help/v3.20/variable/CMAKE_FIND_ROOT_PATH.html#variable:CMAKE_FIND_ROOT_PATH) entries by default in all cases, as well as looking in the host system root prefix. Although this can be controlled on a case-by-case basis, when cross-compiling, it can be useful to exclude looking in either the host or the target for particular artifacts. Generally, includes, libraries and packages should be found in the target system prefixes, whereas executables which must be run as part of the build should be found only on the host and not on the target. This is the purpose of the `CMAKE_FIND_ROOT_PATH_MODE_*` variables.



#### 交叉编译 for Cray-Linux-环境

Cross compiling for compute nodes in the Cray Linux Environment can be done without needing a separate toolchain file. Specifying `-DCMAKE_SYSTEM_NAME=CrayLinuxEnvironment` on the CMake command line will ensure that the appropriate build settings and search paths are configured. The platform will pull its configuration from the current environment variables and will configure a project to use the compiler wrappers from the Cray Programming Environment's `PrgEnv-*` modules if present and loaded.

The default configuration of the Cray Programming Environment is to only support static libraries. This can be overridden and shared libraries enabled by setting the `CRAYPE_LINK_TYPE` environment variable to `dynamic`.

Running CMake without specifying [`CMAKE_SYSTEM_NAME`](https://cmake.org/cmake/help/v3.20/variable/CMAKE_SYSTEM_NAME.html#variable:CMAKE_SYSTEM_NAME) will run the configure step in host mode assuming a standard Linux environment. If not overridden, the `PrgEnv-*` compiler wrappers will end up getting used, which if targeting the either the login node or compute node, is likely not the desired behavior. The exception to this would be if you are building directly on a NID instead of cross-compiling from a login node. If trying to build software for a login node, you will need to either first unload the currently loaded `PrgEnv-*` module or explicitly tell CMake to use the system compilers in `/usr/bin` instead of the Cray wrappers. If instead targeting a compute node is desired, just specify the [`CMAKE_SYSTEM_NAME`](https://cmake.org/cmake/help/v3.20/variable/CMAKE_SYSTEM_NAME.html#variable:CMAKE_SYSTEM_NAME) as mentioned above.



#### 使用Clang交叉编译

某些编译器比如Clang天生是交叉编译器。当使用这些编译器的编译的时候，可以通过`CMAKE_<LANG>_COMPILER_TARGET`的值向编译器传递参数：

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(triple arm-linux-gnueabihf)

set(CMAKE_C_COMPILER clang)
set(CMAKE_C_COMPILER_TARGET ${triple})
set(CMAKE_CXX_COMPILER clang++)
set(CMAKE_CXX_COMPILER_TARGET ${triple})
```

而且，有些编译器并没有在其中集成支持程序（比如连接器），但提供了一种方式来向编译器传递外部工具链的路径。可以在toolchain文件中通过`CMAKE_<LANG>_COMPILER_EXTERNAL_TOOLCHAIN`变量来向编译器传递该路径。



#### 交叉编译 for QNX

作为一个Clang编译器，QNX QCC编译器天生是一个交叉编译器。因此可以通过`CMAKE_<LANG>_COMPILER_TRAGET`的值向编译器传递参数：

```cmake
set(CMAKE_SYSTEM_NAME QNX)

set(arch gcc_ntoarmv7le)

set(CMAKE_C_COMPILER qcc)
set(CMAKE_C_COMPILER_TARGET ${arch})
set(CMAKE_CXX_COMPILER QCC)
set(CMAKE_CXX_COMPILER_TARGET ${arch})

set(CMAKE_SYSROOT $ENV{QNX_TARGET})
```



#### 交叉编译 for Windows CE 

Cross compiling for Windows CE requires the corresponding SDK being installed on your system. These SDKs are usually installed under `C:/Program Files (x86)/Windows CE Tools/SDKs`.

A toolchain file to configure a Visual Studio generator for Windows CE may look like this:

```
set(CMAKE_SYSTEM_NAME WindowsCE)

set(CMAKE_SYSTEM_VERSION 8.0)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_GENERATOR_TOOLSET CE800) # Can be omitted for 8.0
set(CMAKE_GENERATOR_PLATFORM SDK_AM335X_SK_WEC2013_V310)
```

The [`CMAKE_GENERATOR_PLATFORM`](https://cmake.org/cmake/help/v3.20/variable/CMAKE_GENERATOR_PLATFORM.html#variable:CMAKE_GENERATOR_PLATFORM) tells the generator which SDK to use. Further [`CMAKE_SYSTEM_VERSION`](https://cmake.org/cmake/help/v3.20/variable/CMAKE_SYSTEM_VERSION.html#variable:CMAKE_SYSTEM_VERSION) tells the generator what version of Windows CE to use. Currently version 8.0 (Windows Embedded Compact 2013) is supported out of the box. Other versions may require one to set [`CMAKE_GENERATOR_TOOLSET`](https://cmake.org/cmake/help/v3.20/variable/CMAKE_GENERATOR_TOOLSET.html#variable:CMAKE_GENERATOR_TOOLSET) to the correct value.



#### 交叉编译 for Windows 10 Universal Applications

A toolchain file to configure a Visual Studio generator for a Windows 10 Universal Application may look like this:

```
set(CMAKE_SYSTEM_NAME WindowsStore)
set(CMAKE_SYSTEM_VERSION 10.0)
```

A Windows 10 Universal Application targets both Windows Store and Windows Phone. Specify the [`CMAKE_SYSTEM_VERSION`](https://cmake.org/cmake/help/v3.20/variable/CMAKE_SYSTEM_VERSION.html#variable:CMAKE_SYSTEM_VERSION) variable to be `10.0` to build with the latest available Windows 10 SDK. Specify a more specific version (e.g. `10.0.10240.0` for RTM) to build with the corresponding SDK.



#### 交叉编译 for Windows Phone

A toolchain file to configure a Visual Studio generator for Windows Phone may look like this:

```
set(CMAKE_SYSTEM_NAME WindowsPhone)
set(CMAKE_SYSTEM_VERSION 8.1)
```



#### 交叉编译 for Windows Store

A toolchain file to configure a Visual Studio generator for Windows Store may look like this:

```
set(CMAKE_SYSTEM_NAME WindowsStore)
set(CMAKE_SYSTEM_VERSION 8.1)
```



#### 交叉编译 for Android

可以在toolchain文件中设定`CMAKE_SYSTEM_NAME`变量的值为`Android`来设置Android交叉编译。然后还需设置的是所使用的Android开发环境。

For [Visual Studio Generators](https://cmake.org/cmake/help/v3.20/manual/cmake-generators.7.html#visual-studio-generators), CMake expects [NVIDIA Nsight Tegra Visual Studio Edition](https://cmake.org/cmake/help/v3.20/manual/cmake-toolchains.7.html#cross-compiling-for-android-with-nvidia-nsight-tegra-visual-studio-edition) or the [Visual Studio tools for Android](https://cmake.org/cmake/help/v3.20/manual/cmake-toolchains.7.html#cross-compiling-for-android-with-the-ndk) to be installed. See those sections for further configuration details.

对于 `Makefile Generators`和`Ninja Generators，CMake可使用下面的环境之一：

+ NDK
+ Standalone Toolchain

CMake通过下面的步骤来选取一种环境：

+ 如果`CMAKE_ANDROID_NDK`变量被设置，将使用NDK环境。
+ 否则，如果`CMAKE_ANDROID_STANDALONE_TOOLCHAIN`变量被设置，将使用Standalone Toolchain。
+ 否则，如果`CMAKE_SYSROOT`变量被设置为一个路径，且形如`<ndk>/platforms/android-<api>/arch-<arch>`，那么将使用NDK环境，而且`<ndk>`部分将作为`CMAKE_ANDROID_NDK`的值。
+ 否则，如果`CMAKE_SYSROOT`变量被设置为一个路径，且形如`<standalon-toolchain>/sysroot`，那么将使用Standalone Toolchain，而且`<standalone-toolchain>`部分将作为`CMAKE_ANDROID_STANDALONE_TOOLCHAIN`的值。
+ 否则，如果设置了一个名为`ANDROID_NDK`的cmake变量，将使用NDK环境，且用该变量的值作为`CMAKE_ANDROID_NDK`的值。
+ 否则，如果设置了一个名为`ANDROID_STANDALONE_TOOLCHAIN`的cmake变量，将使用Standalone Toolchain，且用该变量的值作为`CMAKE_ANDROID_STANDALONE_TOOLCHAIN`的值。
+ 否则，如果设置了一个名为`ANDROID_NDK_ROOT`或`ANDROID_NDK`的环境变量，将使用NDK环境，且环境变量的值作为`CMAKE_ANDROID_NDK`的值。
+ 否则，如果设置了一个名为`ANDROID_STANDALONE_TOOLCHAIN`的环境变量，将使用Standalone Toolchain，且环境变量的值作为`CMAKE_ANDROID_STANDALONE_TOOLCHAIN`的值。
+ 否则，将报告错误。NDK和Standalone Toolchain都找不到。

##### 使用NDK交叉编译 for Android

A toolchain file may configure [Makefile Generators](https://cmake.org/cmake/help/v3.20/manual/cmake-generators.7.html#makefile-generators), [Ninja Generators](https://cmake.org/cmake/help/v3.20/manual/cmake-generators.7.html#ninja-generators), or [Visual Studio Generators](https://cmake.org/cmake/help/v3.20/manual/cmake-generators.7.html#visual-studio-generators) to target Android for cross-compiling.

通过下面的变量来配置Android的NDK环境：

+ `CMAKE_SYSTEM_NAME`

  设置为`Android`。必须设置为这个值以使启用Android交叉编译。

+ `CMAKE_SYSTEM_VERSION`

  设置Android API Level。如果没有设置，将通过下面的步骤决定其值：

  + 如果`CMAKE_ANDROID_API`变量设置了，将使用该变量的值。
  + 如果`CMAKE_SYSROOT`变量设置了，将从NDK目录结构中探测sysroot得到API Level。
  + 否则，将使用NDK中最近可用的API Level。

+ `CMAKE_ANDROID_ARCH_ABI`

  设置Android ABI（处理器架构）。如果没有设置，默认值将会是`armeabi`、`armeabi-v7a`、`arm64-v8a`中第一个被支持的。`CMAKE_ANDROID_ARCH`变量的值将会从`CMAKE_ANDROID_ARCH_ABI`自动计算出来。另外，关注`CMAKE_ANDROID_ARM_MODE`和`CMAKE_ANDROID_ARM_NEON`变量。

+ `CMAKE_ANDROID_NDK`

  设置Android NDK根目录的绝对路径。

+ `CMAKE_ANDROID_NDK_DEPRECATED_HEADERS`

  Set to a true value to use the deprecated per-api-level headers instead of the unified headers. If not specified, the default will be false unless using a NDK that does not provide unified headers.

+ `CMAKE_ANDROID_NDK_TOOLCHAIN_VERION`

  在NDK r19及以上，这个变量必须不设置或设为`clang`。在NDK r18及以下，设置这个变量的值为NDK工具链的版本来选择编译器。如果没有设置，默认将会是最近可用的GCC工具链。

+ `CMAKE_ANDROID_STL_TYPE`

  Set to specify which C++ standard library to use. If not specified, a default will be selected as described in the variable documentation.

下面的变量将会被自动计算出：

+ `CMAKE_<LANG>_ANDROID_TOOLCHAIN_PREFIX`

  The absolute path prefix to the binutils in the NDK toolchain

+ `CMAKE_<LANG>_ANDROID_TOOLCHAIN_SUFFIX`

  The host platform suffix of the binutils in the NDK toolchain.

这里是一个示例，一个toolchain文件可能如此：

```cmake
set(CMAKE_SYSTEM_NAME Android)
set(CMAKE_SYSTEM_VERSION 21) # API level
set(CMAKE_ANDROID_ARCH_ABI arm64-v8a)
set(CMAKE_ANDROID_NDK /path/to/android-ndk)
set(CMAKE_ANDROID_STL_TYPE gnustl_static)
```

你也可以不通过toolchain文件来指定这些变量：

```shell
$ cmake ../src \
  -DCMAKE_SYSTEM_NAME=Android \
  -DCMAKE_SYSTEM_VERSION=21 \
  -DCMAKE_ANDROID_ARCH_ABI=arm64-v8a \
  -DCMAKE_ANDROID_NDK=/path/to/android-ndk \
  -DCMAKE_ANDROID_STL_TYPE=gnustl_static
```



##### 使用Standalone Toolchain来交叉编译 for Android

 





