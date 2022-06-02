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
>>**xilinx_setup_2021.2-zcu702.sh**
>>```
>>#setup XILINX_VITIS and XILINX_VIVADO variables
>>source /opt/Xilinx/Vitis/2021.2/settings64.sh
>>#setup XILINX_XRT
>>source /opt/xilinx/xrt/setup.sh
>>unset LD_LIBRARY_PATH
>>#source /opt/Xilinx/Vitis/2021.2/data/emulation/qemu/unified_qemu_v5_0/environment-setup-aarch64-xilinx-linux
>>#available platforms for use with your Vitis IDE
>>export PLATFORM_REPO_PATHS=/opt/xilinx/platforms/xilinx_zcu102_base_202120_1
>>#export LD_LIBRARY_PATH=/opt/Xilinx/Vitis/2020.2/lib/lnx64.o/Default
>>export ROOTFS=/opt/Xilinx/xilinx-zynqmp-common-v2021.2
>>export SYSROOT=/opt/petalinux/2021.2/sysroots #/cortexa72-cortexa53-xilinx-linux/
>>source /opt/petalinux/2021.2/environment-setup-cortexa72-cortexa53-xilinx-linux
>>```
> **2. SW Emulation**
>> To build for software emulation, enter the following commands to setup the target build directory:
>> ```
>> cd /local/jinxu/Vitis-Tutorials/Getting_Started/Vitis/example/zcu102
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
>>A brief explanation of these four commands refers to [Build and Run the Embedded Processor Application](https://github.com/Xilinx/Vitis-Tutorials/blob/2021.2/Getting_Started/Vitis/Part4-embedded_platform.md)
>>
>>After the build process completes, launch the software emulation (start the Xilinx Quick Emulation (QEMU) environment and initiate the boot sequence):
>>
>>`./package/launch_sw_emu.sh -forward-port 1440 22`
>>
>>Once Linux has finished booting, you will see following comments
>>```
>>root@zynqmp-common-2021_2:~# xinit: giving up 
>>xinit: unable to connect to X server: Connection refused
>>xinit: server error
>>Enabling notebook extension jupyter-js-widgets/extension...
>>      - Validating: OK
>>[C 12:38:18.687 NotebookApp] Bad config encountered during initialization: No such notebook dir: ''/usr/share/example-notebooks''
>>```
>>Then enter the following commands from within the QEMU environment to run the example program:
>>```
>>cd /media/sd-mmcblk0p1
>>export XILINX_XRT=/usr
>>export XCL_EMULATION_MODE=sw_emu
>>./app.exe
>>```
>>The following messages indicate that the run completed successfully:
>>```
>>INFO: Found Xilinx Platform
>>INFO: Loading 'vadd.xclbin'
>>TCP Port :7717
>>tcp client connected to the server
>>```

>>The above files that were created during this exercise:
>>```
>>root@zynqmp-common-2021_2:/media/sd-mmcblk0p1# ls
>>Image  a.xclbin  app.exe  boot.scr  data  opencl_trace.csv  run_sw_emu.sh  summary.csv	vadd.xclbin  xrt.ini  xrt.run_summary
>>```
>>
>>+ app.exe: The compiled and linked host application
>>
>>+ vadd.xclbin: The device binary linking the kernel and target platform
>>
>>+ opencl_trace.csv: A report of events occurring during the application runtime
>>
>>+ summary.csv: A report of the application profile
>>
>>+ xrt.ini: The runtime initilization file
>>
>>+ xrt.run_summary: A summary report of the events of the application runtime
>>
>>Terminate QEMU:`shudown -h now` 
>>
>>or open another termimal, check the process:`ps aux` and then:`kill -9 <QEMU_port>`
>>
> **3. HW Emulation**
> 
>>Before HW emulation, if you terminated the previous terminal, set up the environment again.
>>
>>```source /opt/Xilinx/xilinx_setup_2021.2-zcu702.sh```
>>
>>To build for hardware emulation, enter the following commands to setup the target build directory:
>>```
>>cd /local/jinxumkdir /Vitis-Tutorials/Getting_Started/Vitis/example/zcu102
>>mkdir hw_emu
>>cp xrt.ini hw_emu
>>cp run_hw_emu.sh hw_emu
>>cd hw_emu
>>```
>>Enter the following commands to build the host application and device binary:
>>```
>>$CXX -Wall -g -std=c++11 ../../src/host.cpp -o app.exe -I/usr/include/xrt -lOpenCL -lpthread -lrt -lstdc++ -O
>>v++ -c -t hw_emu --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm -k vadd -I../../src ../../src/vadd.cpp -o vadd.xo
>>v++ -l -t hw_emu --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm ./vadd.xo -o vadd.xclbin
>>v++ -p -t hw_emu --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm ./vadd.xclbin --package.out_dir package --package.rootfs ${ROOTFS}/rootfs.ext4 --package.sd_file ${ROOTFS}/Image --package.sd_file xrt.ini --package.sd_file app.exe --package.sd_file vadd.xclbin --package.sd_file ./run_hw_emu.sh
>>```
>>The only difference with Software Emulation is the v++ target `-t` option which is changed from `sw_emu` to `hw_emu`. All other options remain the same.
>>
>>Building for hardware emulation takes more time than for software emulation, but still much less than when targeting the hardware accelerator card. 
>>
>>After the build process completes, you can launch the hardware emulation run by using the launch script generated during the packaging step:
>>`./package/launch_hw_emu.sh `
>>
>>Once Linux has finished booting, enter the following commands at the QEMU command prompt to run the example program:
>>```
>>cd /media/sd-mmcblk0p1
>>export XILINX_XRT=/usr
>>export XCL_EMULATION_MODE=hw_emu
>>./app.exe
>>```
>>You should see messages that say TEST PASSED indicating that the run completed successfully
>>
>>Running the application in the QEMU generates some report files during the run. These files and reports are the results of the run process targeting the software emulation build. To examine these files later, we must retrieve them from the QEMU environment and copy them into our local system, for example: `scp -P 1440 root@127.0.0.1:/media/sd-mmcblk0p1/xrt.run_summary ./xrt.run_summary`
>>
>>Terminate QEMU:`shudown -h now` 
>>
>>or open another termimal, check the process:`ps aux` and then:`kill -9 <QEMU_port>`
> **4. Targeting Hardware**
> (no necessary)
>>To build for the hardware target, enter the following commands to setup the target build directory:
>>```
>>cd <Path to the cloned repo>/Getting_Started/Vitis/example/zcu102
>>mkdir hw
>>cp xrt.ini hw
>>cp run_hw.sh hw
>>cd hw
>>```
