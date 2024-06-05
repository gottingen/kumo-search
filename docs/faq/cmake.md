cmake
==========================

# Q: 如何让cmake优先搜索指定路径下的库？

A: 有多种方法可以让cmake优先搜索指定路径下的库：
    
方法1：

    使用`CMAKE_PREFIX_PATH`变量，例如：
    cmake -DCMAKE_PREFIX_PATH=/opt/EA/inf ..

方法2：

    使用HINTS选项，例如：
    find_package(protobuf REQUIRED HINTS /opt/EA/inf)

# Q: 如何让cmake编译的静态库带有-fPIC选项？

A: 有多种方法可以让cmake编译的静态库带有-fPIC选项：

方法1：

    使用`CMAKE_POSITION_INDEPENDENT_CODE`变量，例如：
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)

方法2：

    编译时指定-fPIC选项，例如：
    cmake .. -DCMAKE_CXX_FLAGS=-fPIC

方法3： 针对单个库进行设置

    target_properties(${TARGET_NAME} PROPERTIES POSITION_INDEPENDENT_CODE ON)
    