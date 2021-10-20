# [Simulink]生成C++动态库

- MATLAB版本：R2018b
- Visual Studio版本：2017
- 操作系统：Windows 10

## 1. 创建一个Simulink工程

1. 以Simulink起始页里面的工程为例，在`Embedded Coder`选项卡下，打开第一项：
   ![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/01Simlulink/start_page.png?token=AFQ6BQPFR6ATDDASAOCFLK3BN6X7O)
   ![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/01Simlulink/outlook.png?token=AFQ6BQIZB4HEVJ24RQUHKN3BN6YWS) 

2. 可选：`Ctrl+S`保存工程，命名为Model。

## 2. C++ Code Generation配置

1. 在菜单栏依次打开`Code`>`C/C++ Code` >`Code Generation Options`，并选中`Code Generation`选项卡：
   ![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/01Simlulink/code_generation.png?token=AFQ6BQJ4VAHG6D7IAHICV3LBN62BM)
2. 右侧面板`Target Selection`里面，选择`System target file`为`ert.tlc Create Visual C/C++ Solution File for Embedded Coder`，如图：
   ![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/01Simlulink/system_target.png?token=AFQ6BQNOT3IXG2NGIITPC4DBN62VW)
   这里两个*ert.tlc*，区别在于第一个会直接生成C++的h、cpp源文件，并编译成exe，不会生成VS工程。不过其源代码也很方便就可以嵌入到其他工程或者生成动态库了。
   第二个不仅会生成C++的h文件和cpp文件，还会生成VS工程，所以我们可以直接用VS打开了，便于后面的修改和编译！

3. 同上一步，右侧面板`Target Selection`里面，选择`Language`为`C++`，如图：
   ![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/01Simlulink/language.png?token=AFQ6BQKQ4YFFB5JJND6DTXTBN64BC)

4. 可选：在`Code Generation`的`Interface`子选项卡里面，右侧面板`Code Interface`里面，点击`Configuration C++ Class Interface`，打开如下页面修改步长函数接口和类名：
   ![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/01Simlulink/interface.png?token=AFQ6BQNQL5NOMPQN3URJP5DBN66V2)

## 3. 生成VS工程

`Ctrl+B`执行构建，生成VS工程。工程生成结束后，Visual Studio 2017会自动打开VS工程。如图：
![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/01Simlulink/build.png?token=AFQ6BQJOYBO7IRUKXGHIP6DBN67VU)

## 4. 配置生成DLL

1. 在VS IDE里面打开Model.h文件，定位到48行，并修改为下图：
   ![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/01Simlulink/class_export.png)
   后面我们会在该工程添加`MODEL_LIBRARY`宏定义，所以54行就会展开为`class __declspec(dllexport) MyModel`（49行是灰的，是因为这里还没有定义MODEL_LIBRARY宏)，也就是将MyModel类符号导出，作为动态库供其他程序使用。

2. 打开MyModel工程的属性页，依次执行一下操作：
   - 编译为DLL动态库：
     ![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/01Simlulink/out_dll.png)
   - 添加宏定义：
     ![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/01Simlulink/lib_def.png)
   - 生成动态库：
     ![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/01Simlulink/build_dll.png)

## 5. 动态库的使用

1. VS工程执行玩结束后会生成如下的动态库文件和MyModel类源文件：
   ![](https://raw.githubusercontent.com/Mud-Player/MudPic/main/01Simlulink/out_inter.png)

2. 对该动态库的调用按照标准C++编译链接步骤即可，代码调用示例如下：

   ```
     MyModel model;
     model.initialize;
     model.rtU.In1 = 100;
     model.update();
     std::cout << model.rtY.Out1;
     model.terminate();
   ```

## 6. VS工程优化

Matlab生成的VS工程默认只有一个Debug版本，所以我们需要在工程属性页修改编译选项，以适应自己项目实际的Debug或者Release情况，解决运行速度与编译报错等问题。

通常我们需要修改这些地方：

- C/C++ > 常规 > 优化
- C/C++ > 常规 > 预处理器 > 预处理器宏定义
- 链接器 > 调试 > 生成调试信息
- 链接器 > 优化 > 链接时间代码生成

