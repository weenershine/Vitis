# OpenCL编程详细解析

OpenCL作为一门开源的异构并行计算语言，设计之初就是使用一种模型来模糊各种硬件差异。
作为软件开发人员，关注的就是编程模型。

Vitis 核开发套件支持 OpenCL 1.2 API，如 https://www.khronos.org/registry/OpenCL/specs/
opencl-1.2.pdf 中所述。如需了解有关 OpenCL 的 XRT 扩展的说明，请访问 https://xilinx.github.io/XRT/2020.2/html/
opencl_extension.html。

总开发流程如下：

![image](https://user-images.githubusercontent.com/49140300/188286801-d314191b-8ede-4f50-bf35-f6cf8cd7b58f.png)

OpenCL程序的流程大致如下：

0. Include
1. Platform
2. Device
3. Sub-device
4. Context
5. CommandQeueu
6. Program (for FPGA)
7. Running time
8. 加载 OpenCL 内核程序并创建一个 program 对象
9. 为指定的 device 编译 program 中的 kernel
10. 创建指定名字的 kernel 对象
11. 为 kernel 创建内存对象
12. 为 kernel 设置参数
13. 在指定的 device 上创建 command queue
14. 将要执行的 kernel 放入 command queue
15. 将结果读回 host
16. 资源回收

## 0. Include

使用 OpenCL API 编程与一般 C/C++ 引入第三方库编程没什么区别。
要引入 include 相关的头文件。

`#include "xcl2.hpp"`

## 1. Platform

首先要取得系统中所有的 OpenCL platform。
所谓的 platform 指的就是硬件厂商提供的 OpenCL 框架，
不同的 CPU/GPU/FPGA 开发商（比如 Xilinx、Intel、AMD、Nvdia）可以在一个系统上分别定义自己的 OpenCL 框架。
所以需要查询系统中可用的 OpenCL 框架，即 platform。

OpenCL API 调用 clGetPlatformIDs 用于发现给定系统的可用 OpenCL 平台组合。
随后，clGetPlatformInfo 用于识别基于赛灵思器件的平台，方法是将 cl_platform_vendor 与字符串 "Xilinx" 相匹配。


```
cl_platform_id platform_id; // platform id
err = clGetPlatformIDs(16, platforms, &platform_count);
// Find Xilinx Platform
for (unsigned int iplat=0; iplat<platform_count; iplat++) {
err = clGetPlatformInfo(platforms[iplat],
CL_PLATFORM_VENDOR,
1000,
(void *)cl_platform_vendor,
NULL);
if (strcmp(cl_platform_vendor, "Xilinx") == 0) {
// Xilinx Platform found
platform_id = platforms[iplat];
}
}
```

## 2. Device

找到赛灵思平台后，应用需识别对应的赛灵思器件。clGetDeviceIDs API 通过 platform_id 和 CL_DEVICE_TYPE_ACCELERATOR 来调用，以接收
所有可用的赛灵思器件。
```
cl_device_id devices[16]; // compute device id
char cl_device_name[1001];
err = clGetDeviceIDs(platform_id, CL_DEVICE_TYPE_ACCELERATOR,
16, devices, &num_devices);
printf("INFO: Found %d devices\n", num_devices);
//iterate all devices to select the target device.
for (uint i=0; i<num_devices; i++) {
err = clGetDeviceInfo(devices[i], CL_DEVICE_NAME, 1024, cl_device_name,
0);
printf("CL_DEVICE_NAME %s\n", cl_device_name);
}
```

## 3. Sub-device

在 Vitis 核开发套件中，有时器件包含单一内核或不同内核的多个内核实例。虽然 OpenCL API
clCreateSubDevices 允许主机代码将器件分割为多个子器件，但 Vitis 核开发套件仅支持均等分割的子器件（使用
CL_DEVICE_PARTITION_EQUALLY），且每个子器件包含一个内核实例。

虽然 OpenCL 支持可包含多个器件和子器件的上下文，但 XRT 要求每个器件和子器件都具有独立的上下文。

```
cl_uint num_devices = 0;
cl_device_partition_property props[3] = {CL_DEVICE_PARTITION_EQUALLY,1,0};
// Get the number of sub-devices
clCreateSubDevices(device,props,0,nullptr,&num_devices);
// Container to hold the sub-devices
std::vector<cl_device_id> devices(num_devices);
// Second call of clCreateSubDevices
// We get sub-device handles in devices.data()
clCreateSubDevices(device,props,num_devices,devices.data(),nullptr);
// Iterating over sub-devices
std::for_each(devices.begin(),devices.end(),[kernel](cl_device_id sdev) {
// Context for sub-device
auto context = clCreateContext(0,1,&sdev,nullptr,nullptr,&err);
// Command-queue for sub-device
auto queue = clCreateCommandQueue(context,sdev,
CL_QUEUE_OUT_OF_ORDER_EXEC_MODE_ENABLE,&err);
// Execute the kernel on the sub-device using local context and
queue run_cu(context,queue,kernel); // Function not shown
});
```

## 4. Context
clCreateContext API 用于创建上下文环境，其中包含将与主机进行通信的赛灵思器件。

It is like a backstorage which save all the related information, 
and in which to run the calculation.

`context = clCreateContext(0, 1, &device_id, NULL, NULL, &err);`

在上述代码中，clCreateContext API 用于创建包含 1 个赛灵思器件的上下文环境。赛灵思建议仅为每个器件或每
个子器件创建一个上下文环境。但如果使用多个子器件并且每个子器件对应一个上下文环境，那么主机程序应使用多个
上下文环境。


## 5. CommandQueue

对于openCL，要控制在GPU中内核函数的运行时序，需要用到openCL的命令队列，
在openCL中，每个可执行的设备，其中的工作项执行都由一个命令队列来控制，
命令队列中有多个等待执行的命令，每个命令(Command)有自己的状态机，
每个执行命令可以使用相应的事件来做同步和异步处理。

>命令状态机
>+ CL_QUEUED: 命令已经插入到队列中，等待调度
>
>+ CL_SUBMITTED: 命令已经被调度提交到设备中，等待执行
>
>+ CL_RUNNING: 命令正在执行
>
>+ CL_COMPLETE: 命令执行完成
>
>+ ERROR_CODE: 命令执行有错误

clCreateCommandQueue API 会为每个器件创建一个或多个命令队列。
FPGA 可包含多个内核，这些内核可能相同，也可能不同。
开发主机应用时，有2种主要的编程方法可用于在器件上执行内核。

1. 单个无序命令队列：可通过同一命令队列请求多次内核执行。XRT会尽快以任意顺序分派内核，以允许在 FPGA
上并发执行内核。

`
commands = clCreateCommandQueue(context, device_id, CL_QUEUE_OUT_OF_ORDER_EXEC_MODE_ENABLE, &err);
`

2. 多个有序命令队列：从不同的有序命令队列请求每次内核执行。
在此类情况下，XRT会分派来自不同命令队列的内核，通过在器件上并发运行这些内核来提升性能。

`commands = clCreateCommandQueue(context, device_id, 0, &err);`

## 6. Program (for FPGA)

主机与内核代码分别进行编译，以创建独立的可执行文件：
即主机程序可执行文件和 FPGA 二进制文件 (.xclbin)。
当运行主机应用时，它必须使用 clCreateProgramWithBinary API 加载 .xclbin 文件。

```
unsigned char *kernelbinary;
char *xclbin = argv[1]; //从命令行实参 argv[1] 传入内核二进制文件 .xclbin
printf("INFO: loading xclbin %s\n", xclbin);
int size=load_file_to_memory(xclbin, (char **) &kernelbinary);
//load_file_to_memory 函数用于在主机存储器空间中加载文件内容
size_t size_var = size;
cl_program program = clCreateProgramWithBinary(context, 1, &device_id,
&size_var,(const unsigned char **) &kernelbinary,
&status, &err); 
//[clCreateProgramWithBinary](https://registry.khronos.org/OpenCL/sdk/1.2/docs/man/xhtml/clBuildProgram.html) API 用于在指定上下文和器件中完成程序创建进程。
// Function
int load_file_to_memory(const char *filename, char **result)
{
uint size = 0;
FILE *f = fopen(filename, "rb");
if (f == NULL) {
*result = NULL;
return -1; // -1 means file opening fail
}
fseek(f, 0, SEEK_END);
size = ftell(f);
fseek(f, 0, SEEK_SET);
*result = (char *)malloc(size+1);
if (size != fread(*result, sizeof(char), size, f)) {
free(*result);
return -2; // -2 means file reading fail
fclose(f);
(*result)[size] = 0;
return size;
}
```






















