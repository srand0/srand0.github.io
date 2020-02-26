---
layout: article
title:  'Cartographer(¡)!'
tags: Cartographer Cmake
aside:
  toc: true
mathjax: true
mathjax_autoNumber: true
cover: /image/posts/2020-02-26/1.cartographer-logo.png
---

<!-- ![cartographer-logo](/image/posts/2020-02-26/1.cartographer-logo.png) -->

cartographer是google开源的使用图优化解决SLAM问题的程序。本文先分析cartographer的原始代码结构，然后对代码结构和cmake文件进行简化。

<!--more-->

## 1. 原始结构分析

![structure-origin](/image/posts/2020-02-26/2.structure-origin.png)

cartographer按照官网介绍进行安装即可，此处不进行说明，安装后，可以看到cartographer的代码结构如上所示(build是我自己设置的编译目录)：

- .bazelci：与源码无关，不需要保留
- .github：同上
- cartographer：源代码目录
- cmake：其他模块编译需要的cmake文件目录
- configuration_files：cartographer运行时需要的配置文件
- docs：cartographer文档目录
- scripts：各种脚本，与源码编译无关

下面，我们需要关注的是cartographer目录和cmake目录：

cartographer源码部分目录如下图所示：

![sources](/image/posts/2020-02-26/3.sources.png)

源码目录中包含以下几种文件：
- 源码文件：\*.h为头文件、\*.cc为源文件
- 测试文件：以_test.cc结尾的文件，或者fake、mock开头的文件，或者包含test_helpers的文件
- proto文件：\*.proto文件都是protobuf文件，需要在编译时调用protoc编译成\*.pb.h和\*.pb.cc文件
- grpc文件：cloud目录下的所有文件为需要依赖grpc才能编译的源码文件，但是cartographer核心代码并不需要grpc与外部通信，所以可以移除。

cmake目录中包含以下文件：

![sources](/image/posts/2020-02-26/4.cmakes.png)

- modules：目录中为各个模块对应的cmake文件
- functions：提供google-test等cmake函数

下面进行cmake文件的解析，cmake文件包含以下几个部分：
- 版本声明：cmake开头部分就是版本号的声明
```cmake
set(CARTOGRAPHER_VERSION...
set(CARTOGRAPHER_SOVERSION...
...
``` 

- 依赖包处理：
```cmake
find_package(Abseil...
find_package(Boost...
find_package(Ceres...
...
```
其中，abseil的编译时会自动从github下载源码。


- 源文件收集：
```cmake
file(GLOB_RECURSE ALL_LIBRARY_HDRS "cartographer/*.h")    //获得cartographer目录下的所有.h文件路径，保存到ALL_LIBRARY_HDRS这个变量中
file(GLOB_RECURSE ALL_LIBRARY_SRCS "cartographer/*.cc")   //同上
....
list(REMOVE_ITEM ALL_LIBRARY_SRCS ${ALL_TESTS})   //从ALL_LIBRARY_SRCS中去掉所有的测试文件
....
```
从cmake文件中，可以看到，获得了所有的\*.h和\*.cc文件后，会从里面去掉所有的测试文件、grpc文件等。


- proto编译：
```cmake
add_custom_command(
    OUTPUT "${PROJECT_BINARY_DIR}/${DIR}/${FIL_WE}.pb.cc"
           "${PROJECT_BINARY_DIR}/${DIR}/${FIL_WE}.pb.h"
    COMMAND  ${PROTOBUF_PROTOC_EXECUTABLE}
    ARGS --cpp_out  ${PROJECT_BINARY_DIR} -I ${PROJECT_SOURCE_DIR} ${ABS_FIL}
    DEPENDS ${ABS_FIL}
    COMMENT "Running C++ protocol buffer compiler on ${ABS_FIL}"
    VERBATIM
)
```
proto编译的核心cmake语句如上所示，add_custom_command会调用COMMAND指定的${PROTOBUF_PROTOC_EXECUTABLE}命令，即protoc，将所有的.proto文件编译为.pb.h和.pb.cc文件。


- 生成最终文件：
```cmake
add_library(${PROJECT_NAME} STATIC ${ALL_LIBRARY_HDRS} ${ALL_LIBRARY_SRCS})
...
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC  "${Boost_INCLUDE_DIRS}")
...
target_link_libraries(${PROJECT_NAME} PUBLIC ${Boost_LIBRARIES})
...
```
生成最终的cartographer.a库文件主要的cmake语句如上所示，add_library指定需要生成的目标文件(cartographer)，target_include_directories指定需要包含的头文件目录，target_link_libraries指定需要的依赖库文件。



## 2. 简化

从上面的分析，我们可以看出，cartographer的原始代码还是有一些文件对于编译核心的库文件是没有用的，因此，我们在此将其进行简化，简化分为以下两个部分：
- 去除不需要的文件
- 简化cmake文件

最终cartographer简化后的目录如下所示：

![structure-origin](/image/posts/2020-02-26/4.final.png)

我将cartographer的原始代码搬到了origin目录下，cartographer目录中只保留编译核心代码库所需的文件，另外proto文件也保留在其中；其他类似测试文件、cloude目录全部都移到了useless目录中；cartographer原始代码中的main文件都移到了main目录中；原先的configuration_files目录重命名为config；cmake中保留了各个模块的cmake编译文件，functions文件中的函数不再需要，所以删除了。

简化后的cmake文件如下所示：

``` cmake

cmake_minimum_required(VERSION 3.2)
project(cartographer)

set(CMAKE_CXX_STANDARD 11)

# cmake include
include(${PROJECT_SOURCE_DIR}/cmake/function.cmake)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules)

# abseil
find_package(Abseil REQUIRED)
# boost
find_package(Boost REQUIRED COMPONENTS iostreams)
# ceres
find_package(Ceres REQUIRED COMPONENTS SuiteSparse)
# eigen3
find_package(Eigen3 REQUIRED)
# lua google
find_package(LuaGoogle REQUIRED)
# protobuf
find_package(Protobuf 3.0.0 REQUIRED)
# cairo
include(FindPkgConfig)
PKG_SEARCH_MODULE(CAIRO REQUIRED cairo>=1.12.16)

# set configure directory and generate config.h file
set(CARTOGRAPHER_CONFIGURATION_FILES_DIRECTORY ${PROJECT_SOURCE_DIR}/config CACHE PATH ".lua configuration files directory")
configure_file(${PROJECT_SOURCE_DIR}/cartographer/common/config.h.cmake ${PROJECT_SOURCE_DIR}/generate/cartographer/common/config.h)

# build proto
build_proto_files(cartographer generate PROTO_SRCS)

# include directories
include_directories(${PROJECT_SOURCE_DIR})
include_directories(${PROJECT_SOURCE_DIR}/generate)

# get headers and sources
file(GLOB_RECURSE CARTO_SRCS "cartographer/*.*")

# targets
add_library(${PROJECT_NAME} STATIC ${CARTO_SRCS} ${PROTO_SRCS})

# include
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC "${EIGEN3_INCLUDE_DIR}")
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC "${CERES_INCLUDE_DIRS}")
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC "${LUA_INCLUDE_DIR}")
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC "${Boost_INCLUDE_DIRS}")
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC ${PROTOBUF_INCLUDE_DIR})
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC "${CAIRO_INCLUDE_DIRS}")

# link
target_link_libraries(${PROJECT_NAME} PUBLIC ${EIGEN3_LIBRARIES})
target_link_libraries(${PROJECT_NAME} PUBLIC ${CERES_LIBRARIES})
target_link_libraries(${PROJECT_NAME} PUBLIC ${LUA_LIBRARIES})
target_link_libraries(${PROJECT_NAME} PUBLIC ${Boost_LIBRARIES})
target_link_libraries(${PROJECT_NAME} PUBLIC gflags glog pthread)
target_link_libraries(${PROJECT_NAME} PUBLIC ${PROTOBUF_LIBRARY} standalone_absl)
target_link_libraries(${PROJECT_NAME} PUBLIC ${CAIRO_LIBRARIES})


```

其中build_proto_files是我修改后的用于编译proto文件的函数，源码如下所示：

``` cmake 


# build proto
# build single proto file
function(build_proto ABS_FIL OUT_DIR ABS_FIL_HDR ABS_FIL_SRC)
	# get file path: DIR/FIL_WE.proto
	file(RELATIVE_PATH REL_FIL ${PROJECT_SOURCE_DIR} ${ABS_FIL})
	get_filename_component(DIR ${REL_FIL} DIRECTORY)
	get_filename_component(FIL_WE ${REL_FIL} NAME_WE)
	message("${PROJECT_SOURCE_DIR}/${OUT_DIR}/${DIR}/${FIL_WE}.pb.h")
	set(${ABS_FIL_HDR} "${PROJECT_SOURCE_DIR}/${OUT_DIR}/${DIR}/${FIL_WE}.pb.h" PARENT_SCOPE)
	set(${ABS_FIL_SRC} "${PROJECT_SOURCE_DIR}/${OUT_DIR}/${DIR}/${FIL_WE}.pb.cc" PARENT_SCOPE)
	# build header & source files
	add_custom_command(
			OUTPUT "${PROJECT_SOURCE_DIR}/${OUT_DIR}/${DIR}/${FIL_WE}.pb.cc" "${PROJECT_SOURCE_DIR}/${OUT_DIR}/${DIR}/${FIL_WE}.pb.h"
			COMMAND ${PROTOBUF_PROTOC_EXECUTABLE} ARGS --cpp_out ${PROJECT_SOURCE_DIR}/${OUT_DIR} -I ${PROJECT_SOURCE_DIR} ${ABS_FIL}
			DEPENDS ${ABS_FIL}
			COMMENT "Running C++ protocol buffer compiler on ${ABS_FIL}"
			VERBATIM
	)
endfunction()

# build proto directories
function(build_proto_files INPUT_DIR OUTPUT_DIR RESULT)
	file(GLOB_RECURSE ALL_PROTOS "${INPUT_DIR}/*.proto")
	set(ALL_PROTO_BUILD)
	foreach (ABS_FIL ${ALL_PROTOS})
		build_proto(${ABS_FIL} ${OUTPUT_DIR} ABS_FIL_HDR ABS_FIL_SRC)
		list(APPEND ALL_PROTO_BUILD ${ABS_FIL_SRC})
		list(APPEND ALL_PROTO_BUILD ${ABS_FIL_HDR})
	endforeach ()
	set_source_files_properties(${ALL_PROTO_BUILD} PROPERTIES GENERATED TRUE)
	set(${RESULT} ${ALL_PROTO_BUILD} PARENT_SCOPE)
endfunction()


```

---


