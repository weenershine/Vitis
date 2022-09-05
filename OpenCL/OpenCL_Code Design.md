source 主要由两个文件构成:
+ xxxxx.cl (核函数)
+ main.cpp (主函数)

>*Kernel：它指的是FPGA侧的加速器。这一术语应该是源自OpenCL的说法，任何用来加速的代码，FPGA IP，或者CUDA，都被称为kernel并作为OpenCL框架的调度的对象。
>
>*Host：指的是Zynq架构下的嵌入式CPU，或者加速卡通过PCIe连接的CPU。

**main.cpp:

>To load and control PL kernels from the host application, the execution model contains following steps:
>
>1. Get the OpenCL platform and device, prepare a context and command queue. Program the XCLBIN file and get kernel objects from the program.adf::registerXRT() is still needed, but the device handle can be converted from the XCL domain to XRT domain.
>2. Prepare device buffers for the kernels. Transfer data from host memory to global memory in device.
>3. The host program sets up the kernel with its input parameters and triggers the execution of the kernel on the Versal™ device.
Wait for kernel completion.
>4. Transfer data from global memory in the device back to host memory.
Host code continues processing using the new data in the host memory.

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

>*argc是命令行总的参数个数，由编译器自动计算。
>
>*argv[]是参数，其中argv[0]是程序的全名，后面的参数是命令行后面跟的用户输入的参数。


4. OpenCL相关变量初始化 

```
    std::string binaryFile = argv[1];
    // compute the size of array in bytes
    size_t size_in_bytes = DATA_SIZE * sizeof(int);
    cl_int err;
    cl::CommandQueue q;
    cl::Kernel krnl_vector_add;
    cl::Context context;
```

>* binaryFile 是命令输入`<vector_addition XCLBIN>`，即vector_addition.cl的二进制文件
>*size_t是一些C/C++标准在stddef.h中定义的,size_t 类型表示C中任何对象所能达到的最大长度,它是无符号整数。

5. 设置主机代码存储

```
    // Creates a vector of DATA_SIZE elements with an initial value of 10 and 32
    vector<int, aligned_allocator<int> > source_a(DATA_SIZE, 10);
    vector<int, aligned_allocator<int> > source_b(DATA_SIZE, 32);
    vector<int, aligned_allocator<int> > source_results(DATA_SIZE);
```

> `vector<int, aligned_allocator<int> >` 用于内存对齐，才能实现并行化加速。参考文章[1](https://zhuanlan.zhihu.com/p/349413376)，[2](https://zhuanlan.zhihu.com/p/93824687)。

6. 查找devices, 为每个device创建一个context和commandqueue

```
// The get_xil_devices will return vector of Xilinx Devices
    auto devices = xcl::get_xil_devices();
// read_binary_file() is a utility API which will load the binaryFile
// and will return the pointer to file buffer.
    auto fileBuf = xcl::read_binary_file(binaryFile);
    cl::Program::Binaries bins{{fileBuf.data(), fileBuf.size()}};
    bool valid_device = false;
    for (unsigned int i = 0; i < devices.size(); i++) {
        auto device = devices[i];
        // Creating Context and Command Queue for selected Device
        OCL_CHECK(err, context = cl::Context(device, nullptr, nullptr, nullptr, &err));
        OCL_CHECK(err, q = cl::CommandQueue(context, device, CL_QUEUE_PROFILING_ENABLE, &err));
        
        ... ...
        
        }
```
>*CL_QUEUE_PROFILING_ENABLE: Enable or disable profiling of commands in
the command-queue. If set, the profiling of commands is enabled. Otherwise profiling of commands is disabled (用于分析数据，可收集计时信息) 参考[1](https://www.codenong.com/11763963/)

7. 创建program

```
    for (unsigned int i = 0; i < devices.size(); i++) {
       
        ... ...
        
        std::cout << "Trying to program device[" << i << "]: " << device.getInfo<CL_DEVICE_NAME>() << std::endl;
        cl::Program program(context, {device}, bins, nullptr, &err);
        if (err != CL_SUCCESS) {
            std::cout << "Failed to program device[" << i << "] with xclbin file!\n";
        } else {
            std::cout << "Device[" << i << "]: program successful!\n";
            // This call will extract a kernel out of the program we loaded in the
            // previous line. A kernel is an OpenCL function that is executed on the
            // FPGA. This function is defined in the src/vetor_addition.cl file.
            OCL_CHECK(err, krnl_vector_add = cl::Kernel(program, "vector_add", &err));
            valid_device = true;
            break; // we break because we found a valid device
        }
        }
```

8. 如没有可加载Bins的device，报错

```
    if (!valid_device) {
        std::cout << "Failed to program any device found, exit!\n";
        exit(EXIT_FAILURE);
    }

```


9. 一些关于buffer的commands

```
    // These commands will allocate memory on the FPGA. The cl::Buffer objects can
    // be used to reference the memory locations on the device. The cl::Buffer
    // object cannot be referenced directly and must be passed to other OpenCL
    // functions.
    OCL_CHECK(err, cl::Buffer buffer_a(context, CL_MEM_USE_HOST_PTR | CL_MEM_READ_ONLY, size_in_bytes, source_a.data(),
                                       &err));
    OCL_CHECK(err, cl::Buffer buffer_b(context, CL_MEM_USE_HOST_PTR | CL_MEM_READ_ONLY, size_in_bytes, source_b.data(),
                                       &err));
    OCL_CHECK(err, cl::Buffer buffer_result(context, CL_MEM_USE_HOST_PTR | CL_MEM_WRITE_ONLY, size_in_bytes,
                                            source_results.data(), &err));
```

10. 设置内核参数

```
    // set the kernel Arguments
    int narg = 0;
    OCL_CHECK(err, err = krnl_vector_add.setArg(narg++, buffer_result));
    OCL_CHECK(err, err = krnl_vector_add.setArg(narg++, buffer_a));
    OCL_CHECK(err, err = krnl_vector_add.setArg(narg++, buffer_b));
    OCL_CHECK(err, err = krnl_vector_add.setArg(narg++, DATA_SIZE));
```

11. 存储传输

```
    // These commands will load the source_a and source_b vectors from the host
    // application and into the buffer_a and buffer_b cl::Buffer objects. The data
    // will be be transferred from system memory over PCIe to the FPGA on-board
    // DDR memory.
    OCL_CHECK(err, err = q.enqueueMigrateMemObjects({buffer_a, buffer_b}, 0 /* 0 means from host*/));

    // Launch the Kernel
    OCL_CHECK(err, err = q.enqueueTask(krnl_vector_add));

    // The result of the previous kernel execution will need to be retrieved in
    // order to view the results. This call will write the data from the
    // buffer_result cl_mem object to the source_results vector
    OCL_CHECK(err, err = q.enqueueMigrateMemObjects({buffer_result}, CL_MIGRATE_MEM_OBJECT_HOST));
    q.finish();
```

12. 在主机端获得结果

```
    int match = 0;
    printf("Result = \n");
    for (int i = 0; i < DATA_SIZE; i++) {
        int host_result = source_a[i] + source_b[i];
        if (source_results[i] != host_result) {
            printf(error_message.c_str(), i, host_result, source_results[i]);
            match = 1;
            break;
        } else {
            printf("%d ", source_results[i]);
            if (((i + 1) % 16) == 0) printf("\n");
        }
    }

    std::cout << "TEST " << (match ? "FAILED" : "PASSED") << std::endl;
    return (match ? EXIT_FAILURE : EXIT_SUCCESS);
```
