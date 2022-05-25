
#Readme (Not working yet)

Reference: https://github.com/Xilinx/Vitis-Tutorials/blob/2021.2/Getting_Started/Vitis/Part4-embedded_platform.md <br/>
Preparation: clone the tutorial folder (https://github.com/Xilinx/Vitis-Tutorials) <br/>

##1. Connect the cao1 server.<br/>
[selee@osito (Fedora 34) ~]$ ssh -Y cairn-cao1.irisa.fr <br/>

##2. Set up the environment for ZCU102 board.<br/>
-bash-4.2$ source /opt/Xilinx/xilinx_setup_2021.2-zcu702.sh <br/>
X
##3. Target SW emulation.<br/>
###3.1. Copy files<br/>
cd /local/selee/Git/Vitis-Tutorials/Getting_Started/Vitis/example/zcu102 <br/>
```
mkdir sw_emu 
cp xrt.ini sw_emu
cp run_sw_emu.sh sw_emu
cd sw_emu
```
###3.2. Build host application and device binary <br/>
current path: /local/selee/Git/Vitis-Tutorials/Getting_Started/Vitis/example/zcu102/ <br/> 

N.B: Unlikely to data center card, it is mandatory to specify --platform option. <br/>
```

$CXX -Wall -g -std=c++11 ../../src/host.cpp -o ./app.exe -I/usr/include/xrt -lOpenCL -lpthread -lrt -lstdc++ 

v++ -c -t sw_emu --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm -k vadd -I../../src ../../src/vadd.cpp -o ./vadd.xo 

v++ -l -t sw_emu --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm ./vadd.xo -o ./vadd.xclbin 

v++ -p -t sw_emu --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm ./vadd.xclbin --package.out_dir ./package --package.rootfs ${ROOTFS}/rootfs.ext4 --package.sd_file ${ROOTFS}/Image --package.sd_file ./xrt.ini --package.sd_file ./app.exe --package.sd_file ./vadd.xclbin --package.sd_file ./run_sw_emu.sh 
```

###3.3 Launch the software emulation run
current path: /local/selee/Git/Vitis-Tutorials/Getting_Started/Vitis/example/zcu102/sw_emu <br/> 

Launch the software emulation run and note the qemu_pid to kill QEMU later.

```
./package/launch_sw_emu.sh -forward-port 1440 22
```

Once Linux has finished booting, you will see following comments.
root@zynqmp-common-2021_2:~# Enabling notebook extension jupyter-js-widgets/extension...
      - Validating: OK
xinit: giving up
xinit: unable to connect to X server: Connection refused
xinit: server error
[C 13:27:16.755 NotebookApp] Bad config encountered during initialization: No such notebook dir: ''/usr/share/example-notebooks''

###3.4 Run the program
Enter the following commands from within the QEMU environment to run the example program
```
cd /media/sd-mmcblk0p1
export XILINX_XRT=/usr
export XCL_EMULATION_MODE=sw_emu
./app.exe
```

Once run completed successfully following comments are shown.
INFO: Found Xilinx Platform
INFO: Loading 'vadd.xclbin'
TEST PASSED

###4.5 Terminate QEMU

open a new terminal and write a follwoing command

kill -9 <qemu_pid>

NB: if a keyboard doesn't work properly, terminate the terminal and run ssh and set up the environment again. 

##4. Target HW emulation
###4.1 Copy files
cd /local/selee/Git/Vitis-Tutorials/Getting_Started/Vitis/example/zcu102
mkdir hw_emu
cp xrt.ini hw_emu
cp run_hw_emu.sh hw_emu
cd hw_emu

###4.2 Build the host application and device binary
N.B: in the official tutorial, run_app is written but this is typo and it should be replace in run_hw_emu.sh.
current path: /local/selee/Git/Vitis-Tutorials/Getting_Started/Vitis/example/zcu102/hw_emu <br/> 

$CXX -Wall -g -std=c++11 ../../src/host.cpp -o app.exe -I/usr/include/xrt -lOpenCL -lpthread -lrt -lstdc++

v++ -c -t hw_emu --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm -k vadd -I../../src ../../src/vadd.cpp -o vadd.xo 

v++ -l -t hw_emu --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm ./vadd.xo -o vadd.xclbin

v++ -p -t hw_emu --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm ./vadd.xclbin --package.out_dir package --package.rootfs ${ROOTFS}/rootfs.ext4 --package.sd_file ${ROOTFS}/Image --package.sd_file xrt.ini --package.sd_file app.exe --package.sd_file vadd.xclbin --package.sd_file run_hw_emu.sh

Once building has completed, you can see following information.

INFO: [v++ 60-2460] Successfully copied a temporary xclbin to the output xclbin: /local/selee/Git/Vitis-Tutorials/Getting_Started/Vitis/example/zcu102/hw_emu/a.xclbin
INFO: [v++ 60-2343] Use the vitis_analyzer tool to visualize and navigate the relevant reports. Run the following command. 
    vitis_analyzer ./v++.package_summary 
INFO: [v++ 60-791] Total elapsed time: 0h 1m 9s
INFO: [v++ 60-1653] Closing dispatch client.

###4.3 Launch the hardware emulation
Launch the hardware emulation run and note the qemu_pid to kill QEMU later.

./package/launch_hw_emu.sh 

In this case, qemu_pid = 13956 is shown in the terminal

Once Linux has finished booting, you will see following comments.
Enabling notebook extension jupyter-js-widgets/extension...
      - Validating: OK

PetaLinux 2021.2 zynqmp-common-2021_2 ttyPS0

root@zynqmp-common-2021_2:~# xinit: giving up
xinit: unable to connect to X server: Connection refused
xinit: server error
[C 14:42:13.797 NotebookApp] Bad config encountered during initialization: No such notebook dir: ''/usr/share/example-notebooks''

###4.4 Run the example program
Enter the following commands from within the QEMU environment to run the example program

cd /media/sd-mmcblk0p1
export XILINX_XRT=/usr
export XCL_EMULATION_MODE=hw_emu
./app.exe

Once run completed successfully following comments are shown.

INFO: Found Xilinx Platform
INFO: Loading 'vadd.xclbin'
TEST PASSED

###4.5 Terminate QEMU

open a new terminal and write a follwoing command

kill -9 <qemu_pid>

NB: if letters written by a keyboard doesn't show up, terminate the terminal and run ssh and set up the environment again. 

##5. Target HW

###5.1 set up the environment
If you terminated the previous terminal, set up the environment again.

source /opt/Xilinx/xilinx_setup_2021.2-zcu702.sh
cd /local/selee/Git/Vitis-Tutorials/Getting_Started/Vitis/example/zcu102

###5.2 Copy files

cd /local/selee/Git/Vitis-Tutorials/Getting_Started/Vitis/example/zcu102
mkdir hw
cp xrt.ini hw
cp run_hw.sh hw
cd hw

###5.3 Build the host application and device binary
current path: /local/selee/Git/Vitis-Tutorials/Getting_Started/Vitis/example/zcu102/hw

$CXX -Wall -g -std=c++11 ../../src/host.cpp -o app.exe -I/usr/include/xrt -lOpenCL -lpthread -lrt -lstdc++ -O

v++ -c -t hw --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm -k vadd -I../../src ../../src/vadd.cpp -o vadd.xo

v++ -l -t hw --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm ./vadd.xo -o vadd.xclbin

v++ -p -t hw --config ../../src/zcu102.cfg --platform $PLATFORM_REPO_PATHS/xilinx_zcu102_base_202120_1.xpfm ./vadd.xclbin --package.out_dir package --package.rootfs ${ROOTFS}/rootfs.ext4 --package.sd_file ${ROOTFS}/Image --package.sd_file xrt.ini --package.sd_file app.exe --package.sd_file vadd.xclbin --package.sd_file run_hw.sh

N.B: in the official tutorial, run_app is written but this is typo and it should be replace in run_hw.sh.

##6. Setting a ZCU102 board
 
###6.1 Get sd card files  
 
 
 Mode SW6 [4:1] mode3: off, mode2: off, mode1: off, mode0: on
 J7 connect a jumper to use a usb peripheral.
