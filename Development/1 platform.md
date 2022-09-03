# Vitis 2021.2软件平台
对于基于FPGA的加速，Vitis核心开发工具包允许:
1. 使用API构建软件应用程序，比如建立一个软件应用程序的OpenCL™ API，
2. 运行硬件（HW）内核上加速卡，如赛灵思 Alveo数据中心加速卡。
3. 支持嵌入式处理器平台上运行Linux和软件应用，如 Versal
VCK190 平台、Zynq® UltraScale+™ MPSoC ZCU102/ZCU104 基本平台以及 Zynq-7000 基本平台
4. **对于嵌入式处理器平台，Vitis核心开发套件执行模型还使用OpenCL API和基于Linux的Xilinx 运行时（XRT），用于调度硬件内核并控制数据移动。**
5. 除了这些现成的平台以外，还支持自定义平台。

通过Vitis软件平台，可以将数据中心应用程序迁移到嵌入式平台。所述Vitis核心开发工具包括在V++台上的硬件内核编译器，g++编译器用于编译在x86主机上运行的应用，以及ARM®用于交叉编译应用程序到的嵌入式处理器上运行的编译器的Xilinx设备。

## 1.2 迁移
1.2.1 从SDAccel迁移
![image](https://user-images.githubusercontent.com/49140300/188268739-14af7b97-a68e-4546-8383-df1d8f495555.png)

1.2.2 从SDSoC迁移

[**SDSoC™ 开发环境**](https://china.xilinx.com/products/design-tools/legacy-tools/sdsoc.html)
可为异构 Zynq® SoC 及 MPSoC 部署提供类似嵌入式 C/C++/OpenCL 应用的开发体验，
+ Xilinx OpenCV 库包含 50 多项硬件优化 OpenCV 功能，包括 Gausian、Median、Bilateral、Harris corner、Canny edge detection、HoG、SVM、LK Optical Flow 及更多
+ 简单易用的 Eclipse IDE 可用于开发支持嵌入式 C/C++/OpenCL 应用的完整 Zynq SoC 和 MPSoC
+ 只需一点按钮，就可对可编程逻辑 (PL) 中的功能进行加速
+ 支持作为目标 OS 的裸机、Linux 与 FreeRTOS
+ 快速性能估算与面积估算可在几分钟内完成，包括 PS、数据通信以及 PL
+ 高速缓存、存储器以及总线利用率的自动运行时仪表
+ 可实现最佳总体系统架构的便捷生成与探索
+ 可将 C/C++/OpenCL 应用编译成全功能 Zynq SoC 与 MPSoC 系统
+ 可在生成 ARM 软件与 FPGA 比特流的可编程逻辑中实现自动功能加速
+ 支持吞吐量、时延以及面积权衡的快速系统探索
![image](https://user-images.githubusercontent.com/49140300/188269107-d89f50a2-64ca-461f-80e7-6267bb1e673e.png)



在Vitis环境中迁移总述：
1. 调用Arm交叉编译器以构建主应用程序代码
2. 调用Vitis编译器以构建硬件内核。
3. 为主机（.elf）创建一个可执行文件，为硬件内核（.xclbin）创建一个映像。
4. 使用OpenCL API和基于Linux的Xilinx运行时（XRT）来控制主应用程序和内核之间的数据移动，并计划任务的执行。

对于嵌入式处理器平台（或项目），Vitis环境仅支持运行Linux主机操作系统且平台已将XRT和ZOCL驱动程序添加到根文件系统的平台。
平台创建者需要提供一个sysroot，以便通过OpenCL包含文件和库交叉编译到Arm 核心。

要从SDSoC环境迁移到Vitis环境，必须修改构建脚本和源代码。
本节讨论迁移步骤，包括命令行示例，这些示例使用sysroot中的文件，使用Vitis编译器编译硬件内核，并使用Arm cross编译器编译主机应用程序。

**基本迁移步骤**

+ 迁移主机应用程序
  + 更新所需的#include文件
  + 根据需求，编辑主要功能以及任何其他软件特定功能
+ 迁移硬件功能
  + 使用编译指示定义内核接口
+ 建立系统
  + 构建和链接内核
  + 编译和链接应用程序
+ 迁移主机应用程序

### EXAMPLE_MMULT

+ 主机应用程序

```
#include <stdlib.h>
#include <iostream>
#include "mmult.h"
#include "sds_lib.h"

#define NUM_TESTS 5

void printMatrix(int *mat, int col, int row) {
 for (int i = 0; i < col; i++) {
  for (int j = 0; j < row; j++) {
   std::cout << mat[i*row+j] << "\t";
  }
  std::cout << std::endl;
 }
 std::cout << std::endl;
}

int main() {
 int col = BUFFER_SIZE;
 int row = BUFFER_SIZE;
 int *matA = (int*)sds_alloc(col*row*sizeof(int));
 int *matB = (int*)sds_alloc(col*row*sizeof(int));
 int *matC = (int*)sds_alloc(col*row*sizeof(int));

 std::cout << "Mat A" << std::endl;
 printMatrix(matA, col, row);

 std::cout << "Mat B" << std::endl;
 printMatrix(matB, col, row);

 //Run the hardware function multiple times
 for(int i = 0; i < NUM_TESTS; i++) {
  std::cout << "Test #: " << i << std::endl;
  // Populate matA and matB
  srand(time(NULL));
  for (int i = 0; i < col*row; i++) {
   matA[i] = rand()%10;
   matB[i] = rand()%10;
  }

  std::cout << "MatA * MatB" << std::endl;
  mmult(matA, matB, matC, col, row);

 }
 printMatrix(matC, col, row);

 return 0;
```
  
该代码为存储为一维数组的三个不同的二维矩阵，分配内存matA，matB使用随机数填充和 ，然后相乘matA并matB计算 matC。结果打印到屏幕上，测试运行十次。
  
+  迁移硬件功能
  
为了在PL中执行，现在将硬件功能单独编译为.xo文件，因此它不包含在main（）函数中，
并且不需要像SDSoC环境中那样的用于函数定义的特定头文件。

在丢弃头文件之前，必须将BUFFER_SIZE声明复制到mmult函数中。（可删除该 #include语句）
 
+ 建立系统

在SDSoC™环境中，sds++编译器同时构建主应用程序（.elf）和硬件加速功能（PL区域的位流）。
在示例中，用于编译main（）函数和mmult硬件函数的命令行如下：

```
sds++ -Wall -O0 -g -I"../src" -c -fmessage-length=0 -MT"src/mmult.o" -MMD \
-MP -MF"src/mmult.d" -MT"src/mmult.o" -o "src/mmult.o" "../src/mmult.cpp" \
-sds-hw mmult mmult.cpp -sds-end -sds-sys-config a53_linux -sds-proc      \
a53_linux -sds-pf "zcu102"
```

从该单个命令中，该sds++命令处理该sds-hw块以编译mmult函数，
然后再链接main.o对象文件以构建目标应用程序main.elf。
最后，它将生成sd_card文件夹，其中包含启动和运行应用程序所需的适当文件。

在Vitis环境中，硬件内核和主要应用程序的编译由两个单独的编译器执行。

## 1.3 支持平台
1.3.1 数据中心加速

1.3.2 嵌入式平台

不支持 Artix®-7，Kintex®-7，Virtex®-7以及裸机和RTOS平台进行加速。
可与Vitis核心开发套件一起使用的嵌入式平台：
+ zcu102_base：基于ZCU102 Zynq UltraScale + MPSoC和XRT。
+ zcu104_base：基于ZCU104 Zynq UltraScale + MPSoC和XRT。
+ zc702_base：基于ZC702Zynq®- 7000SoC 和XRT。
+ **zc706_base：基于ZC706Zynq®- 7000SoC 和XRT。**

