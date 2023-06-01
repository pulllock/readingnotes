# CMakeLists.txt

# 最低版本要求

`CMakeLists.txt`文件第一行必须是最低版本要求：`cmake_minimum_required(VERSION 3.5)`，在CMake 3.12开始版本号可以声明为一个范围：`cmake_minimum_required(VERSION 3.7...3.21)`

```
# 如果CMake版本低于3.12，则CMake将被设置为当前版本
if(${CMAKE_VERSION} VERSION_LESS 3.12)
    cmake_policy(VERSION ${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION})
endif()
```

# 设置项目

`project(MyProject VERSION 1.0 DESCRIPTION "Very nice project" LANGUAGES CXX)`

- 第一个参数MyProject是项目名字
- 第二个参数VERSION 1.0是版本号
- 第三个参数DESCRIPTION "Very nice project"是描述，在CMake 3.9中可以添加描述
- 第四个参数LANGUAGES CXX是语言，语言可以是：C,CXX,Fortran,ASM,CUDA(CMake 3.8+),CSharp(3.8+),SWIFT(CMake 3.15+ experimental)

# 生成可执行文件

`add_executable(one two.cpp three.h)`

- 第一个参数one是要生成的可执行文件的名称，也是创建CMake目标（target）的名字
- 后面的参数是源文件列表

# 生成一个库

`add_library(one STATIC two.cpp three.h)`

- 第一个参数是名字
- 第二个参数STATIC是类型，类型可以是：STATIC、SHARED、MODULE
- 后面的参数是源文件列表

# 目标

- `target_include_directories(one PUBLIC include)`，为目标添加一个目录
- `target_link_libraries`

# 变量

## 本地变量

- `set(MY_VARIABLE "value")`：声明本地变量
- `${MY_VARIABLE}`：引用变量
- `set(MY_LIST "one" "two")`或`set(MY_LIST "one;two")`：包含一系列变量的列表

## 缓存变量

- `set(MY_CACHE_VARIABLE "value" CACHE STRING "Description")`
- `set(MY_CACHE_VARIABLE "value" CACHE INTERNAL "Description")`

## 环境变量

- `set(ENV{variable_name} value)`：设置环境变量
- `$ENV{variable_name}`：获取环境变量

# 缓存

`CMakeCache.txt`