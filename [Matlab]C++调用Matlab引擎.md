# [MATLAB]C++调用MATLAB引擎

- MATLAB版本：R2018b
- 操作系统：Window 10

## 1 需求分析

在C++开发过程中，可能需要C++和MATLAB混合编程，比如：别人只会MATLAB，需要和我们的C++一起运行；或者有部分MATLAB/Simulink开发的模块或者函数，需要快速整合到C++中。那么通常我们会考虑以MATLAB为主程序调用C++代码或者以C++为主程序调用MATLAB代码的方式。

## 2 C++/MATLAB混合编程方式

MATALB官方提供了多种MATLAB与其他语言混合编程的方法，对于C++与MATLAB的混合编程包括下面两种：

1. **C++调用MATLAB**
  - 用于C++的MATLAB引擎API
  - 将MATLAB函数编译为C++动态库

2. **MATLAB调用C/C++**
  - MATLAB调用C++ MEX函数
  - 通过calllib调用C动态库

MATLAB引擎API主要通过matlab::engine::MATLABEngine类和matlab::data::Array等类完成MATLAB的接口功能，其实C++ MEX函数本质上也是调用MATLAB引擎API的接口。后面将介绍以C++最为主程序，调用MATLAB引擎的方法。

## 3 API基础认识

这里先列举常用的类及其常用函数接口：

1. [matlab::engine::MATLABEngine](https://www.mathworks.com/help/releases/R2018b/matlab/apiref/matlab.engine.matlabengine.html)引擎类
  - startMATLAB — 打开一个MATALB引擎
  - connectMATLAB — 连接到一个已共享的MATALB引擎
  - feval — 执行函数
  - eval — 执行指令语句
  - getVariable — 获取变量
  - setVariable — 设置变量
2. [matlab::data::ArrayFactory](https://www.mathworks.com/help/releases/R2018b/matlab/apiref/matlab.data.arrayfactory.html) 矩阵工厂类
  - createArray — 创建数组
  - createScalar — 创建标量
  - createCellArray — 创建元胞数组
  - createCharArray — 创建Char数组
  - createStructArray — 创建结构体数组
3. [matlab::data::Array](https://localhost:31515/static/help/matlab/apiref/matlab.data.array.html)基础数据类型
  - operator — 索引操作
  - getType — 获取数据类型
  - getDimensions — 获取数据维度
  - getNumberOfElements — 获取数据元素个数
  - isEmpty — 数据是否为空

## 4 C++编码开发

### 4.1 环境搭建

1. 安装MATLAB，安装过程略（任何版本都可以，这里以2018b为例），并假定$MATLAB_ROOT作为安装路径，方便后文描述；
2. 安装CMake，这里我主要是方便QtCreator和VS2017以上都可以使用同一个cmake工程。

### 4.2 创建工程

首先创建CMake工程，指定main.cpp文件和MATLAB头文件目录、lib文件：

创建MatlabEngine文件夹，添加CMakeLists.txt文件。粘贴下面代码后执行CMake并确认无报错：

```
cmake_minimum_required(VERSION 3.5)
project(MatlabEngine LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(${PROJECT_NAME} main.cpp)

# 设置Matlab头文件及依赖库
set(MATLAB_ROOT "D:/Program Files/MATLAB/R2018b")
target_include_directories(${PROJECT_NAME} PUBLIC ${MATLAB_ROOT}/extern/include)
target_link_libraries(${PROJECT_NAME} PUBLIC ${MATLAB_ROOT}/extern/lib/win64/microsoft/libMatlabEngine.lib)
target_link_libraries(${PROJECT_NAME} PUBLIC ${MATLAB_ROOT}/extern/lib/win64/microsoft/libMatlabDataArray.lib)
```

注意第9行`set(MATLAB_ROOT "D:/Program Files/MATLAB/R2018b")`，需要改为MATLAB实际的安装路径。

***

注：

对于Windows环境，需要使用到libMatlabEngine和libMatlabDataArray这两个库，对MATLAB引擎的调用实际上是对MATLAB动态库的调用。
qmake可以添加`LIBS += -L$$(MATLAB_ROOT) -l$$(MATLAB_ROOT)/extern/lib/win64/microsoft/libMatlabEngine.lib -l$$(MATLAB_ROOT)/extern/lib/win64/microsoft/libMatlabDataArray.lib)`实现动态库的添加；
vs可以通过界面`1.工程属性 > 链接器 > 工程属性 > 链接器 附加库目录 2.工程属性 > 链接器 > 输入 > 附加依赖项`完成库的添加。

### 4.3 编码

添加main.cpp文件，粘贴最简单的几行代码（将会创建一个figure窗口）：

```
#include <MatlabEngine.hpp>
#include <MatlabDataArray.hpp>

int main()
{
    std::unique_ptr<matlab::engine::MATLABEngine> matlabPtr = matlab::engine::startMATLAB();
    matlabPtr->eval(u"figure");
    return 0;
}
```

### 4.4 编译执行

QtCreator和VS对CMake的执行方式不太一下，这里分开说明：

- QtCreator
  1. 左上角工程名右键点击执行CMake；
  2. 点击左侧`项目`，然后在`构建环境`栏点击`详情`下拉栏，点击`批量编辑`，输入`PATH=${PATH};matlabroot\extern\bin\win64`并点击`OK`以完成添加环境变量。注意这里的matlabroot需要改为MATLAB的实际安装路径，比如：`PATH=${PATH};D:\Program Files\MATLAB\R2018b\extern\bin\win64`。
  3. 左下角点击运行按钮（或Ctrl+R）。

- VS2019

  1. 在解决方案资源管理器窗口点击右上角`在解决方案和可用视图之间切换`，然后选择CMake目标视图；

  2. 选中`MatlabEngine(可执行文件)`后右键点击`添加调试配置`，然后在launch.vs.json文件添加一行`"env": "PATH=$(PATH);matlabroot\\extern\\bin\\win64"`以完成添加环境变量。注意这里的matlabroot需要改为MATLAB的实际安装路径，比如：`"env": "PATH=$(PATH);D:\\Program Files\\MATLAB\\R2018b\\extern\\bin\\win64"`。最终launch.vs.json内容如下：
     ```
     {
       "version": "0.2.1",
       "defaults": {},
       "configurations": [
         {
           "type": "default",
           "project": "CMakeLists.txt",
           "projectTarget": "MatlabEngine.exe",
           "name": "MatlabEngine.exe",
           "env": "PATH=$(PATH);D:\\Program Files\\MATLAB\\R2018b\\extern\\bin\\win64"
         }
       ]
     }
     ```

   3. 工具栏处点击`MatlabEngine.exe`开始运行（或Ctrl+F5）。

***

注：

如果正常运行，将会看到MATLAB生成一个通过figure命令创建的窗口。

MATLAB运行时主要依赖libMatlabEngine.dll和libMatlabDataArray.dll两个动态库，在QtCreator里面，没有找到动态库什么也不会提示，程序会直接崩溃。而VS友好一点，会提示缺少什么库。如果嫌麻烦，可以在电脑的环境变量里面将MATLAB动态库路径添加到Path里面，不过这样所有程序都会受到环境变量的影响，不建议使用。

## 5 应用

1. 创建2行3列的32位整数数组（array_2_3())：

   ```
       matlab::data::ArrayFactory factory;
       matlab::data::TypedArray<int32_t> array = factory.createArray<int32_t>({2, 3});
       array[0][0] = 1;
       array[0][1] = 2;
       array[0][2] = 3;
       array[1][0] = 4;
       array[1][1] = 5;
       array[1][2] = 6;
   ```

2. 将C++的MATLAB数据类型赋值到MATLAB工作区并使用bar命令绘制柱状图(showBar())：

   ```
       m_matlab->setVariable(u"array", array);
       m_matlab->eval(u"bar(array)");
   ```

3. 从MATLAB找到数组最大值并赋值到C++（findMax())：

   ```
       m_matlab->setVariable(u"array", array);
       m_matlab->eval(u"maxValue = max(array, [], 'all');");
       auto tmp = m_matlab->getVariable(u"maxValue");
       int max = tmp[0];
       auto label = new QLabel;
       label->setText(u8"最大值：" + QString::number(max));
       label->show();
   ```

4. 执行.m文件：
   
   4.1 创建mysum.m文件：
   
   ```
   function [output] = mysum(inputArg1,inputArg2)
   output = inputArg1 + inputArg2;
end
   ```
   
   4.2 将mysum.m移动到exe工作路径（不是exe文件路径），或者通过[`path`](https://localhost:31515/static/help/matlab/ref/path.html?searchHighlight=path&searchResultIndex=1)命令添加mysum目录；
   
   4.3 执行.m文件：
   
   ```
       m_matlab->eval(u"sumValue = mysum(1, 2);");
       auto tmp = m_matlab->getVariable(u"sumValue");
       int sum = tmp[0];
       auto label = new QLabel;
       label->setText(u8"两数之和：" + QString::number(sum));
       label->show();
   ```

测试代码地址：https://github.com/Mud-Player/MatlabEngine