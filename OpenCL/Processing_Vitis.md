# OpenCL+Vitis编程详细解析

OpenCL作为一门开源的异构并行计算语言，用来模糊各种硬件差异。
[OpenCl基本流程](https://zhuanlan.zhihu.com/p/445939233)
[OpenCl视频入门](https://www.bilibili.com/video/BV1c54y1Y75t?p=2&vd_source=669f6fea14599939187b680ddc901c20)

Vitis 核开发套件支持 OpenCL 1.2 API，如 https://www.khronos.org/registry/OpenCL/specs/
opencl-1.2.pdf 中所述。如需了解有关 OpenCL 的 XRT 扩展的说明，请访问 https://xilinx.github.io/XRT/2020.2/html/
opencl_extension.html。

总开发流程如下：

![image](https://user-images.githubusercontent.com/49140300/188286801-d314191b-8ede-4f50-bf35-f6cf8cd7b58f.png)


OpenCL+Vitis程序的流程细节如下：

0. Include
1. Platform
2. Device
3. Sub-device
4. Context
5. CommandQeueu
6. Program (for FPGA)
7. Setting Kernel（for FPGA）
8. Buffer Transmission （for FPGA）
9. Kernel Excution
10.加载 OpenCL 内核程序并创建一个 program 对象
11. 为指定的 device 编译 program 中的 kernel
12. 创建指定名字的 kernel 对象
13. 为 kernel 创建内存对象
14. 为 kernel 设置参数
15. 在指定的 device 上创建 command queue
16. 将要执行的 kernel 放入 command queue
17. 将结果读回 host
18. 资源回收

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

## 7. Setting Kernel (for FPGA)

完成 OpenCL 环境初始化后，主机应用即可随时向器件发出命令并与内核进行交互，首先进行内核设置。

设置运行时环境（如识别device、创建context、commandqueue和program）后，主机应用应识别将在device上执行的Kernel并**设置内核实参**。

使用 [OpenCL API clCreateKernel](https://registry.khronos.org/OpenCL/sdk/1.2/docs/man/xhtml/clCreateKernel.html) 访问 .xclbin 文件中所包含的内核（“program”）。cl_kernel 对象用于识别加载到 FPGA 内的程序中的内核，此内核可供主机应用运行。以下代码示例可识别加载的程序中定义的2个内核。

```
kernel1 = clCreateKernel(program, "<kernel_name_1>", &err);
kernel2 = clCreateKernel(program, "<kernel_name_2>", &err); // etc
```

在 Vitis 软件平台中，可为内核对象设置两种类型的实参：
1. 标量实参用于小型数据传输，例如，常量或配置类型数据。从主机应用角度来看，这些实参为只读实参，即内核的输入。
2. 存储缓冲器实参则用于大型数据传输。值是所创建的存储器对象的指针，其context与程序和内核对象相关联。这些实参是内核的输入或输出。

内核实参可使用 [clSetKernelArg](https://registry.khronos.org/OpenCL/sdk/1.2/docs/man/xhtml/clSetKernelArg.html) 命令来设置，以下示例演示了如何为 2 个标量实参和 2 个缓冲器实参设置内核实
参。

```
// Create memory buffers
cl_mem dev_buf1 = clCreateBuffer(context, CL_MEM_WRITE_ONLY |
CL_MEM_USE_HOST_PTR, size, &host_mem_ptr1, NULL);
cl_mem dev_buf2 = clCreateBuffer(context, CL_MEM_READ_ONLY |
CL_MEM_USE_HOST_PTR, size, &host_mem_ptr2, NULL);

int err = 0;
// Setup scalar arguments
cl_uint scalar_arg_image_width = 3840;
err |= clSetKernelArg(kernel, 0, sizeof(cl_uint), &scalar_arg_image_width);
cl_uint scalar_arg_image_height = 2160;
err |= clSetKernelArg(kernel, 1, sizeof(cl_uint),
&scalar_arg_image_height);


// Setup buffer arguments
err |= clSetKernelArg(kernel, 2, sizeof(cl_mem), &dev_buf1);
err |= clSetKernelArg(kernel, 3, sizeof(cl_mem), &dev_buf2);
```

虽然 OpenCL 允许在内核入队前随时设置内核实参，但应尽早设置内核实参。如果在 XRT 确定器件上缓冲器的放置位置之前，移植该缓冲器，那么 XRT 将出错。因此，需在任意缓冲器上执行任意入队操作（例如，clEnqueueMigrateMemObjects）之前设置内核实参。

对于所有内核缓冲器实参，必须在器件全局存储器上**分配缓冲器**。

>*有时，开始内核执行之前无需缓冲器内容。例如，仅在内核执行期间填充输出缓冲器内容，因此内核执行前，这些内容无关紧要。在此情况下，应指定clEnqueueMigrateMemObject（含 CL_MIGRATE_MEM_OBJECT_CONTENT_UNDEFINED 标记），这样缓冲器移植就不涉及主机与器件之间的 DMA 操作，从而即可改善性能。*

## 8. Buffer Transmission

有两种方法可用于分配存储缓冲器和传输数据：
1. 由 XRT 分配缓冲器 (嵌入式主用)
2. 使用主机指针缓冲器

为了最大限度提升从主机到全局存储器的吞吐量，单一缓冲器大小<=4GB 且 >=2MB。

在嵌入式平台上，执行连续存储器分配更高效，在创建缓冲器时交由 XRT 来分配主机存储器。创建缓冲器时，分配是通过使用CL_MEM_ALLOC_HOST_PTR 标记来完成的，随后使用 clEnqueueMapBuffer 将已分配的存储器映射到用户空间指针。

```
// Two cl_mem buffer, for read and write by kernel
cl_mem dev_mem_read_ptr = clCreateBuffer(context, CL_MEM_ALLOC_HOST_PTR |
CL_MEM_READ_ONLY,
sizeof(int) * number_of_words, NULL, NULL);
cl_mem dev_mem_write_ptr = clCreateBuffer(context, CL_MEM_ALLOC_HOST_PTR |
CL_MEM_WRITE_ONLY,
sizeof(int) * number_of_words, NULL, NULL);
cl::Buffer in1_buf(context, CL_MEM_ALLOC_HOST_PTR | CL_MEM_READ_ONLY,
sizeof(int) * DATA_SIZE, NULL, &err);
// Setting arguments
clSetKernelArg(kernel, 0, sizeof(cl_mem), &dev_mem_read_ptr);
clSetKernelArg(kernel, 1, sizeof(cl_mem), &dev_mem_write_ptr);

// Get Host side pointer of the cl_mem buffer object
auto host_write_ptr =
clEnqueueMapBuffer(queue,dev_mem_read_ptr,true,CL_MAP_WRITE,0,bytes,0,nullpt
r,nullptr,&err);
//clEnqueueMapBuffer API 可映射指定的缓冲器，并将 XRT 创建的指针返回到该映射区域
auto host_read_ptr =
clEnqueueMapBuffer(queue,dev_mem_write_ptr,true,CL_MAP_READ,0,bytes,0,nullpt
r,nullptr,&err);
// Fill up the host_write_ptr to send the data to the FPGA
for(int i=0; i< MAX; i++) {
host_write_ptr[i] = <.... >
}
//使用数据填充主机侧指针
// Migrate //进行往来器件的数据传输
cl_mem mems[2] = {host_write_ptr,host_read_ptr};
clEnqueueMigrateMemObjects(queue,2,mems,0,0,nullptr,&migrate_event));
// Schedule the kernel
clEnqueueTask(queue,kernel,1,&migrate_event,&enqueue_event);
// Migrate data back to host
clEnqueueMigrateMemObjects(queue, 1, &dev_mem_write_ptr,
CL_MIGRATE_MEM_OBJECT_HOST,1,&enqueue_event,
&data_read_event);
clWaitForEvents(1,&data_read_event);
// Now use the data from the host_read_ptr
```



>*默认情况下，当内核链接到平台时，来自所有内核的存储器接口都连接到单个默认的全局存储体。因此，每次在该全局存储体上只能有 1 个计算单元 (CU) 执行数据传输，这就限制了应用的总体性能。*
>
>*如果器件仅包含 1 个全局存储体，那么这是唯一选项。但是，如果器件包含多个全局存储体，可以通过在链接期间修改内核的存储器接口连接来自定义全局存储体连接，参考[将内核端口映射到存储器](https://docs.xilinx.com/r/2021.1-Chinese/ug1393-vitis-application-acceleration/%E5%B0%86%E5%86%85%E6%A0%B8%E7%AB%AF%E5%8F%A3%E6%98%A0%E5%B0%84%E5%88%B0%E5%AD%98%E5%82%A8%E5%99%A8)。针对不同内核或计算单元使用不同的独立存储体可以支持多个内核存储器接口进行数据并发读写，从而提升总体性能。*


子缓冲器: 
从器件缓冲器中读取某一特定部分。

## 9. Kernel Excution











