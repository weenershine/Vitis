1. platform

可以理解为不同的（vendor的）OpenCL实现，比如你的电脑中装有AMD显卡和NVIDIA显卡，
那么如果在电脑中安装了AMD的OpenCL SDK和NVIDIA的OpenCL SDK的话，
你的电脑中就包含了两个platform。

2. context

是管理同一platform下的devices，
所以不能创建一个context同时包含AMD和NVIDIA的device。
可以为不同的platform创建context对象。
但是，可以为同一个platform上的devices们创建多个context。

![image](https://user-images.githubusercontent.com/49140300/188319687-09cbe8c7-1b09-488a-88af-2629dce3b501.png)

3. kernel

是运行在device上一个函数，通常这些kernel函数写在.cl文件中。

4. program

是kernel容器，和context联系在一起的。
在创建program对象前，需要将cl文件中代码内容读到内存中，
然后通过clCreateProgrammWithSource
（或者使用kernel的二进制代码通过clCreateProgramWithBinary）来创建program对象。

> OpenCL程序中先由cl文件创建program对象，
> 
> 然后利用program对象再创建kernel对象，
>
> 创建kernel时不需要指定device，
>
> kernel可以被送往context中的任何一个device。

5. command

是host告诉device要进行的操作（device不可以向host发送command）。
注意，操作不光是kernel函数的执行，
还有一些数据传输（data transfer，host->device,device->host）等工作。

6. command queue

负责将command从host发送给device
（负责context与device之间的通讯），
每个command queue对象和context与device是对应的（创建时指定）。
一个command queue中可以包含多个command，
一般情况下是command顺序进行执行的（FIFO），
在创建comamnd queue对象时也可以设置command queue的属性（CL_QUEUE_OUT_OF_ORDER_EXEC_MODE_ENABLE），
让其执行的顺序是乱序的（比如说，一个kernel还没执行完，device就开始了另一个kernel函数的执行）。
