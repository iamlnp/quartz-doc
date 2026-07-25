https://cmake.org/cmake/help/latest/command

# CMake语句



## A

1. **add_executable(hello-world hello-world.cpp)**: 指示CMake创建一个新目标：可执行文件`hello-world`。这个可执行文件是通过编译和链接源文件`hello-world.cpp`生成的
2. **add_library(message STATIC Message.hpp Message.cpp**`：生成必要的构建指令，将指定的源码编译到库中。`add_library`的第一个参数是目标名。生成库是根据第二个参数(`STATIC`或`SHARED`)和操作系统确定的。
   1. **STATIC**：用于创建静态库，即编译文件的打包存档，以便在链接其他目标时使用，例如：可执行文件。
   2. **SHARED**：用于创建动态库，即可以动态链接，并在运行时加载的库。可以在`CMakeLists.txt`中使用`add_library(message SHARED Message.hpp Message.cpp)`从静态库切换到动态共享对象(DSO)。
   3. **OBJECT**：可将给定`add_library`的列表中的源码编译到目标文件，不将它们归档到静态库中，也不能将它们链接到共享对象中。如果需要一次性创建静态库和动态库，那么使用对象库尤其有用。我们将在本示例中演示。
   4. **MODULE**：又为DSO组。与`SHARED`库不同，它们不链接到项目中的任何目标，不过可以进行动态加载。该参数可以用于构建运行时插件。
3. **add_dependencies**：`add_dependencies(<target> [<target-dependency>]...)`, 定义 target 依赖的其他 target，确保在编译该 target 之前，其他 target 已经被构建
4. 

## C

1. **cmake_minimum_required(VERSION 3.5 FATAL_ERROR)**: 设置CMake所需的最低版本。如果使用的CMake版本低于该版本，则会发出致命错误



## F

1. **foreach-endforeach**: 循环语句，举例：

```cmake
foreach(_source IN LISTS sources_with_lower_optimization)
  set_source_files_properties(${_source} PROPERTIES COMPILE_FLAGS -O2)
  message(STATUS "Appending -O2 flag for ${_source}")
endforeach()
```

2. **find_package**：从外部项目查找和加载设置，用于发现和设置包的CMake模块的命令。这些模块包含CMake命令，用于标识系统标准位置中的包。CMake模块文件称为`Find<name>.cmake`，当调用`find_package(<name>)`时，模块中的命令将会运行，除了在系统上实际查找包模块之外，查找模块还会设置了一些有用的变量，反映实际找到了什么。

- 找到Python解释器**find_package(PythonInterp REQUIRED)**，`FindPythonInterp.cmake`附带的设置了一些CMake变量:
  - **PYTHONINTERP_FOUND**：是否找到解释器
  - **PYTHON_EXECUTABLE**：Python解释器到可执行文件的路径
  - **PYTHON_VERSION_STRING**：Python解释器的完整版本信息
  - **PYTHON_VERSION_MAJOR**：Python解释器的主要版本号
  - **PYTHON_VERSION_MINOR** ：Python解释器的次要版本号
  - **PYTHON_VERSION_PATCH**：Python解释器的补丁版本号

- **find_package(PythonLibs ${PYTHON_VERSION_MAJOR}.${PYTHON_VERSION_MINOR} EXACT REQUIRED)**找到Python头文件和库，对应的模块是`FindPythonLibs.cmake`,如果找到，可以使用如下变量，如果python非标准安装，也可以使用-D选项对下列变量修改：
  - **PYTHON_LIBRARY**：指向Python库的路径
  - **PYTHON_INCLUDE_DIR**：Python.h所在的路径

## G

1. **get_source_file_property(VAR file property)**:检索给定文件所需属性的值，并将其存储在CMake`VAR`变量中。```get_source_file_property(_flags ${_source} COMPILE_FLAGS)```
2. 

## I

1. **if(变量) ...else()...endif()**: 条件变量

```cmake
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    message(STATUS "Configuring on/for Linux")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    message(STATUS "Configuring on/for macOS")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    message(STATUS "Configuring on/for Windows")
elseif(CMAKE_SYSTEM_NAME STREQUAL "AIX")
    message(STATUS "Configuring on/for IBM AIX")
else()
    message(STATUS "Configuring on/for ${CMAKE_SYSTEM_NAME}")
endif()
```



## L

1. **list(APPEND _sources Message.hpp Message.cpp**: 引入一个变量`_sources`，包括`Message.hpp`和`Message.cpp`， 生成一个源文件列表

## M

1. **message**:打印消息， 例如：`message("C++ compiler flags: ${CMAKE_CXX_FLAGS}")`

## O

1. **option(<option_variable> "help string" [initial value])**:以选项的形式显示逻辑开关，用于外部设置，从而切换构建系统的生成行为, ``option(USE_LIBRARY "Compile sources into a library" OFF)`, 可以通过CMake的`-D`CLI选项，将信息传递给CMake来切换库的行为:`cmake -D USE_LIBRARY=ON ..`, `-D`开关用于为CMake设置任何类型的变量：逻辑变量、路径等等。
2. 

## P

1. **project(recipe-01 LANGUAGES CXX)**:声明了项目的名称(`recipe-01`)和支持的编程语言(CXX代表C++)

## S

1. **set (USE_LIBRARY OFF)**:引入了一个新变量`USE_LIBRARY`，这是一个逻辑变量，值为`OFF`
1. **set_target_properties**: 设置目标属性

```cmake
set_target_properties(animals
  PROPERTIES
    CXX_STANDARD 14
    CXX_EXTENSIONS OFF
    CXX_STANDARD_REQUIRED ON
    POSITION_INDEPENDENT_CODE 1
)
```

3. **set_source_files_properties(file PROPERTIES property value)**:它将属性设置为给定文件的传递值。与目标非常相似，文件在CMake中也有属性，允许对构建系统进行非常细粒度的控制, `set_source_files_properties(${_source} PROPERTIES COMPILE_FLAGS -O2)`
4. 

## T

1. **target_link_libraries(hello-world message)**: 将库链接到可执行文件。

2. **target_compile_options**：
```cmake
   target_compile_options(compute-areas
     PRIVATE
       "-fPIC"
     )
   ```

​		CMake将编译选项视为目标属性。因此，可以根据每个目标设置编译选项，而不需要覆盖CMake默认值。编译选项可以添加三个级别的可见性：`INTERFACE`、`PUBLIC`和`PRIVATE`。

- **PRIVATE**，编译选项会应用于给定的目标，不会传递给与目标相关的目标。我们的示例中， 即使`compute-areas`将链接到`geometry`库，`compute-areas`也不会继承`geometry`目标上设置的编译器选项。

- **INTERFACE**，给定的编译选项将只应用于指定目标，并传递给与目标相关的目标。

- **PUBLIC**，编译选项将应用于指定目标和使用它的目标。
3. **target_sources**： 指定在构建目标及其依赖项时要使用的源。名为的目标必须由 [add_executable](https://so.csdn.net/so/search?q=add_executable&spm=1001.2101.3001.7020)()、add_library() 或者 add_custom_target() 等命令创建，且不能是 ALIAS 目标。




# CMake命令

1. **cmake --build .**: 指定`CMakeLists.txt`的位置(本例中位于父目录中)来调用CMake, 同`cmake -H. -Bbuild`(`-H`表示当前目录中搜索根`CMakeLists.txt`文件。`-Bbuild`告诉CMake在一个名为`build`的目录中生成所有的文件。)
2. **cmake ..**:配置CMake
3. **CMAKE_<LANG>_COMPILER**:其中`<LANG>`是受支持的任何一种语言，对于我们的目的是`CXX`、`C`或`Fortran`。例如：`cmake -D CMAKE_CXX_COMPILER=clang++ ..`
4. **cmake --system-information information.txt**: CMake提供`--system-information`标志，它将把关于系统的所有信息转储到屏幕或文件中

# 变量

1. **BUILD_SHARED_LIBS**`: 预定义变量，使用shared参数`add_library`

2. **USE_LIBRARY**:使用库方式

3. **CMAKE_BUILD_TYPE**:控制生成构建系统使用的配置变量, CMake识别的值为:

   1. **Debug**：用于在没有优化的情况下，使用带有调试符号构建库或可执行文件。
   2. **Release**：用于构建的优化的库或可执行文件，不包含调试符号。
   3. **RelWithDebInfo**：用于构建较少的优化库或可执行文件，包含调试符号。
   4. **MinSizeRel**：用于不增加目标代码大小的优化方式，来构建库或可执行文件。

   使用举例：`set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)`

4. **CMAKE_CONFIGURATION_TYPES**:变量可以对这些生成器的可用配置类型进行调整，该变量将接受一个值列表, 举例：`cmake .. -G"Visual Studio 12 2017 Win64" -D CMAKE_CONFIGURATION_TYPES="Release;Debug"`

5. **CMAKE_CXX_FLAGS**: 可控制全局编译标记， 例如： `cmake -D CMAKE_CXX_FLAGS="-fno-exceptions -fno-rtti" ..`

6. **CXX_STANDARD**:设置语言标准, 值为11、14等

7. **CXX_EXTENSIONS**: 告诉CMake，只启用`ISO C++`标准的编译器标志，而不使用特定编译器的扩展，值为ON、OFF

8. **CXX_STANDARD_REQUIRED**指定所选标准的版本。如果这个版本不可用，CMake将停止配置并出现错误。当这个属性被设置为`OFF`时，CMake将寻找下一个标准的最新版本，直到一个合适的标志。这意味着，首先查找`C++14`，然后是`C++11`，然后是`C++98`

9. **COMPILE_FLAGS**：编译标记， 比如-O2、-O3等等

10. **CMAKE_SYSTEM_NAME**: 检测操作系统

4. **CMAKE_CXX_COMPILER_ID**: 检测编译器相关， 值为Intel、GNU、PGI、XL等等

```cmake
target_compile_definitions(hello-world PUBLIC "COMPILER_NAME=\"${CMAKE_CXX_COMPILER_ID}\"")
if(CMAKE_CXX_COMPILER_ID MATCHES Intel)
  target_compile_definitions(hello-world PUBLIC "IS_INTEL_CXX_COMPILER")
endif()
```

12. **CMAKE_HOST_SYSTEM_PROCESSOR**：检测当前运行的处理器名称，比如i386、i686、x86_64

```cmake
if(CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "i386")
    message(STATUS "i386 architecture detected")
endif()
```

13. **CMAKE_SIZEOF_VOID_P**:检查当前CPU是否具有32位或64位架构，比如IS_64_BIT_ARCH、IS_32_BIT_ARCH

```cmake
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
  target_compile_definitions(arch-dependent PUBLIC "IS_64_BIT_ARCH")
  message(STATUS "Target is 64 bits")
endif()
```

14. **PYTHON_EXECUTABLE**：Python解释器到可执行文件的路径， **`cmake -D PYTHON_EXECUTABLE=/custom/location/python ..`**, 设置对应的选项。

# 概念

1. **列表**： CMake中，是用分号分隔的字符串组。列表可以由`list`或`set`命令创建。例如，`set(var a b c d e)`和`list(APPEND a b c d e)`都创建了列表`a;b;c;d;e`
2. 



# 函数库

1. **CMakePrintHelpers**: 

```cmake
include(CMakePrintHelpers)
cmake_print_variables(_status _hello_world)
输出：
-- _status="0" ; _hello_world="Hello, world!"
```



# 文件作用

<img src="CMake命令速查/image/image-20220331221909842.png" alt="image-20220331221909842" style="zoom:50%;" />

GNU/Linux上，CMake默认生成Unix Makefile来构建项目：

- `Makefile`: `make`将运行指令来构建项目。
- `CMakefile`：包含临时文件的目录，CMake用于检测操作系统、编译器等。此外，根据所选的生成器，它还包含特定的文件。
- `cmake_install.cmake`：处理安装规则的CMake脚本，在项目安装时使用。
- `CMakeCache.txt`：如文件名所示，CMake缓存。CMake在重新运行配置时使用这个文件。

