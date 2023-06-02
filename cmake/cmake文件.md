# CMakeLists.txt

- `cmake_minimum_required(VERSION 3.5)`：`CMakeLists.txt`文件第一行必须是最低版本要求在CMake，3.12开始版本号可以声明为一个范围：`cmake_minimum_required(VERSION 3.7...3.21)`

```
# 如果CMake版本低于3.12，则CMake将被设置为当前版本
if(${CMAKE_VERSION} VERSION_LESS 3.12)
    cmake_policy(VERSION ${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION})
endif()
```

- `project(project_name VERSION 1.0 DESCRIPTION "Very nice project" LANGUAGES CXX)`：定义项目信息 
  - 第一个参数project_name是项目名字
  - 第二个参数VERSION 1.0是版本号
  - 第三个参数DESCRIPTION "Very nice project"是描述，在CMake 3.9中可以添加描述
  - 第四个参数LANGUAGES CXX是项目支持的语言，语言可以是：C,CXX,Fortran,ASM,CUDA(CMake 3.8+),CSharp(3.8+),SWIFT(CMake 3.15+ experimental)
  - `project()`默认隐式定义了两个cmake变量：`<project_name>_BINARY_DIR`和`<project_name>_SOURCE_DIR`；cmake也预定了两个变量`PROJECT_BINARY_DIR`和`PROJECT_SOURCE_DIR`，值和`<project_name>_BINARY_DIR`和`<project_name>_SOURCE_DIR`是一样的
- `add_executable(one two.cpp three.h)`：生成可执行文件 
  - 第一个参数one是要生成的可执行文件的名称，也是创建CMake目标（target）的名字
  - 后面的参数是源文件列表
- `add_library(one STATIC two.cpp three.h)`：生成一个库
  - 第一个参数是名字
  - 第二个参数STATIC是类型，类型可以是：STATIC、SHARED、MODULE
  - 后面的参数是源文件列表
- `target_include_directories(one PUBLIC include)`：添加一个目录
- `target_link_libraries`
- `set(MY_VARIABLE "value")`：声明本地变量
- `${MY_VARIABLE}`：引用变量
- `set(MY_LIST "one" "two")`或`set(MY_LIST "one;two")`：包含一系列变量的列表
- `set(MY_CACHE_VARIABLE "value" CACHE STRING "Description")`：设置缓存变量
- `set(MY_CACHE_VARIABLE "value" CACHE INTERNAL "Description")`
- `set(ENV{variable_name} value)`：设置环境变量
- `$ENV{variable_name}`：获取环境变量
- `CMakeCache.txt`：缓存
- `message(message_type, "message to display")`：向终端输出信息，message_type包含三种类型：
  - `SEND_ERROR`：产生错误，生成过程被跳过
  - `STATUS`：输出前缀为`-`的信息
  - `FATAL_ERROR`：立即终止所有cmake过程