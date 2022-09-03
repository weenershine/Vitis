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

* Setting Up the Environment to Run the Vitis Software Platform**
>\*To configure the environment to run the Vitis software platform, run the following scripts in a command shell to set up the tools to run in that shell:
>```
>#set up XILINX_VITIS and XILINX_VIVADO variables 
>source /opt/Xilinx/Vitis/2021.2/settings64.sh
>#set up XILINX_XRT for data center platforms (not required for embedded platforms)
>source /opt/xilinx/xrt/setup.sh
>```
>\*To install a platform, download the zip file and extract it into /opt/xilinx/platforms, or extract it into a separate location and add that location to the PLATFORM_REPO_PATHS environment variable.
>`export PLATFORM_REPO_PATHS=/opt/xilinx/platforms/xilinx_zcu102_base_202120_1`
>
Instead of above, we use the command:

`source /opt/Xilinx/xilinx_setup_2021.2-zcu702.sh`
>
>**xilinx_setup_2021.2-zcu702.sh**
>```
>#setup XILINX_VITIS and XILINX_VIVADO variables
>source /opt/Xilinx/Vitis/2021.2/settings64.sh
>#setup XILINX_XRT
>source /opt/xilinx/xrt/setup.sh
>unset LD_LIBRARY_PATH
>#source /opt/Xilinx/Vitis/2021.2/data/emulation/qemu/unified_qemu_v5_0/environment-setup-aarch64-xilinx-linux
>#available platforms for use with your Vitis IDE
>export PLATFORM_REPO_PATHS=/opt/xilinx/platforms/xilinx_zcu102_base_202120_1
>#export LD_LIBRARY_PATH=/opt/Xilinx/Vitis/2020.2/lib/lnx64.o/Default
>export ROOTFS=/opt/Xilinx/xilinx-zynqmp-common-v2021.2
>export SYSROOT=/opt/petalinux/2021.2/sysroots #/cortexa72-cortexa53-xilinx-linux/
>source /opt/petalinux/2021.2/environment-setup-cortexa72-cortexa53-xilinx-linux
>```
