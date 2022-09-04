source 主要由两个文件构成:
+ xxxxx.cl (核函数)
+ main.cpp (主函数)

>Kernel：它指的是FPGA侧的加速器。这一术语应该是源自OpenCL的说法，任何用来加速的代码，FPGA IP，或者CUDA，都被称为kernel并作为OpenCL框架的调度的对象。
>
>Host：指的是Zynq架构下的嵌入式CPU，或者加速卡通过PCIe连接的CPU。

**main.cpp:

1. include头文件

```
#include "xcl2.hpp"
#include <vector>

using std::vector;
```

2. 设置静态常量

```
static const int DATA_SIZE = 1024;
static const std::string error_message =
    "Error: Result mismatch:\n"
    "i = %d CPU result = %d Device result = %d\n";
```

3. 编写main函数基本结构

```
int main(int argc, char** argv) {
    if (argc != 2) {
        std::cout << "Usage: " << argv[0] << " <XCLBIN File>" << std::endl;
        return EXIT_FAILURE;
    }
    
    ......
    ......

}
```

4. 
