# Servers in Team

* start the server
>Vitis (version 2021.2) is now available on cao1.
>For those not targeting the Alveo U280 card, use Vitis on cao1 rather than cao2!
>To use it: source /opt/Xilinx/xilinx_setup_2021.2.sh (or source /opt/Xilinx/xilinx_setup_2021.2.csh for csh shell.
>If your target is a specific platform (ZCU102, ZCU104, ...), let me know and I will install the base platforms for these cards.


* open the vnc
```
vncserver -geometry 1920x1080
vncserver -geometry 2048x1024
```

* workspace
>/local/jinxu


# Build and Run the Embedded Processor Application 
>Ref: [Vitis Getting Started Tutorial](https://github.com/Xilinx/Vitis-Tutorials/blob/2021.2/Getting_Started/Vitis/Part2.md)
>
>Ref: [UG1393](https://docs.xilinx.com/r/2021.2-English/ug1393-vitis-application-acceleration/Setting-Up-the-Environment-to-Run-the-Vitis-Software-Platform)
>
>Ref: [vitisSue](https://github.com/weenershine/Vitis/blob/main/vitisSue.md)
>
> **1. Setting Up the Environment to Run the Vitis Software Platform**
>>\*To configure the environment to run the Vitis software platform, run the following scripts in a command shell to set up the tools to run in that shell:
>>```
>>#set up XILINX_VITIS and XILINX_VIVADO variables 
>>source /opt/Xilinx/Vitis/2021.2/settings64.sh
>>#set up XILINX_XRT for data center platforms (not required for embedded platforms)
>>source /opt/xilinx/xrt/setup.sh
>>```
>>\*To install a platform, download the zip file and extract it into /opt/xilinx/platforms, or extract it into a separate location and add that location to the PLATFORM_REPO_PATHS environment variable.
>>`export PLATFORM_REPO_PATHS=/opt/xilinx/platforms/xilinx_zcu102_base_202120_1`
>>
>>Instead of above, we use the command:
>>`source /opt/Xilinx/xilinx_setup_2021.2-zcu702.sh`
>>
> **2. SW Emulation**
>> To build for software emulation, enter the following commands to setup the target build directory:
>> ```
>> cd /local/jinxu/Git/Vitis-Tutorials/Getting_Started/Vitis/example/zcu102
>> mkdir sw_emu 
>> cp xrt.ini sw_emu
>> cp run_sw_emu.sh sw_emu
>> cd sw_emu
>> ```
>> Then, after changing into the target build directory, enter the following commands to build the host application and device binary:
>> 
>> Unlikely to data center card, it is mandatory to specify --platform option.
>> ```
>> $CXX -Wall -g -std=c++11 ../../src/host.cpp -o ./app.exe -I/usr/include/xrt -lOpenCL -lpthread -lrt -lstdc++ 
>> v++ -c -t sw_emu --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm -k vadd -I../../src ../../src/vadd.cpp -o ./vadd.xo 
>> v++ -l -t sw_emu --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm ./vadd.xo -o ./vadd.xclbin 
>> v++ -p -t sw_emu --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm ./vadd.xclbin --package.out_dir ./package --package.rootfs ${ROOTFS}/rootfs.ext4 --package.sd_file ${ROOTFS}/Image --package.sd_file ./xrt.ini --package.sd_file ./app.exe --package.sd_file ./vadd.xclbin --package.sd_file ./run_sw_emu.sh 
>> ```


* <Vitis_install_path>: 
