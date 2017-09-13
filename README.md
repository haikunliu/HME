# HME:A Lightweight Emulator for Hybrid Memory

&#160; &#160; &#160; &#160; HME is a DRAM-based performance emulator to emulate the performance and energy
characteristics of upcoming NVM technologies. HME exploits features available in commodity NUMA architectures to emulate two kinds of memories: fast, local DRAM, and slower, remote NVM on other NUMA nodes. HME can emulates a wide range of NVM latencies and bandwidth by injecting different memory access delays on the remote NUMA nodes. To help programmers and researchers in evaluating the impact of NVM on the application performance, we also provide a high-level programming interface to allocate memory from NVM or DRAM pools - AHME. 

![Image of Yaktocat](https://github.com/Gumi-presentation-by-Dzh/HME/blob/master/images/architecture.png)

HME has achieved following functions:

 * **Latency Emulation**: As there is no programming interface in commodity hardware to directly control memory access latency, we adopt a software approach to emulate the NVM latency. The basic idea is to inject software-generated additional latencies for the applications in fix-sized time intervals. This strategy can model application-awared total memory access latency to approximate the NVM latency in a period of time. HME uses Home Agent (HA) in Intel Xeon Processor uncore PMU (Performance Monitoring Units) to record the number of memory read accesses (LLC misses) and write accesses to the DRAM on the remote Node, and these memory requests should be issued from applications running on local Node. HME injects additional delays to the cores of the local Node through inter-processor interrupts (IPIs). In such way, HME increases the remote memory access latencies to emulate NVM latency.

![Image of Yaktocat](https://github.com/Gumi-presentation-by-Dzh/HME/blob/master/images/latency.png)


 * **Bandwidth Emulation**: HME emulate NVM bandwidth by limiting the maximum available DRAM bandwidth. The bandwidth throttling is achieved by leveraging a DRAM thermal control interface provided in commodity Intel Xeon processors.

![Image of Yaktocat](https://github.com/Gumi-presentation-by-Dzh/HME/blob/master/images/bw.png)

 * **Energy Emulation**: HME have also established an energy consumption model to compute the NVM’s energy consumption. Due to the lack of the hardware-level feature for counting the energy consumed by the main memory, we choose a statistical approach to model the NVM energy consumption. Unlike the DRAM, NVM does not generate static power consumption, so we only need to count the NVM read and write energy consumption. We count the number of NVM read and write through the PMU and estimate the NVM energy consumption.


HME Setup,Compiling,Configuration and How to use
------------
**1.External Dependencies**  
&#160; &#160; &#160; &#160; Before install hybrid simulator HME, it's essential that you have already install dependencies listing below.
* gcc(>=4.6)
* numactl-devel
* [libconfig](http://www.hyperrealm.com/libconfig/libconfig-1.5.tar.gz) or libconfig-devel
* kernel-devel
* python(>=2.7)
* PMU Toolkit (When you using Multicore version, it needs to be manually installed and configured) [Intel PMU profiling tools](https://github.com/andikleen/pmu-tools.git)

You can run 'sudo /scripts/install.sh' in order to automatically install some of these dependencies.

**2.Compiling**

[![Build Status](https://travis-ci.org/RRZE-HPC/likwid.svg?branch=master)](https://travis-ci.org/RRZE-HPC/likwid)

* Compiling and Installation

First, Compiling the emulator's module. From the emulator's source code /Regular_version/HME folder, execute make.

```javascript
[root @node1 HME]# cd Regular_version
[root @node1 Regular_version]# cd HME
[root @node1 HME]# make  //to compiling the HME
```

* Update configuration to your HME configuration through /Regular_version/HME/scripts/nvmini.in
```javascript
[Latency]:
    DRAM_Read_latency_ns = 100          //DRAM read latency(ns)
    DRAM_Write_latency_ns = 200         //DRAM write latency(ns)
    NVM_Read_latency_ns = 400           //NVM read latency(ns)
    NVM_Write_latency_ns = 1800         //NVM write latency(ns)
[Bandwith]: 
    NVM_bw_ratio = 2                    //One proportion of Bandwidth of DRAM / Bandwidth of NVM
[Model]:
    epoch_duration_us = 100             //Simulator polling time
    type = 1                            //when type =0 it means you using the lib AHME; When type=1 it means you using libnuma to put all in nvm; When type=2 it means you using libnuma policy interleave to put it in DRAM and NVM
[Consumption]:
    NVM_read_w = 100                    //NVM read consumption
    NVM_write_w = 2000                  //NVM write consumption
```

* Use HME through /Regular_version/HME/scripts/runenv.sh
```javascript
[root @node1 scripts]# sh runenv.sh runspec --config=Example-linux64-amd64-gcc43.cfg --noreportable --iteration=1 433.milc
```
Using runenv.sh to run your command , it will be put in the HME to emulation the hybrid memory.

**3.Mult_core version**
Make sure your PMU-TOOL is working without error.
/Multcore_version/HME/run.sh is used to start this tool.
/Multcore_version/HME/core_NVM.c  is used to realize the driver core_NVM.ko which is used to receive the performance delta of every core and send these information to every core
/Multcore_version/HME/delay_count.py is used to calculate the performance delta of every core
/Multcore_version/HME/NVM_emulate_bandwidth is used to control the nvm bandwith, it same as regular_version we describe above.

AHME：PROGRAMMING INTERFACE
------------

&#160; &#160; &#160; &#160; We describe the programming interfaces of HME, named AHME, which can help the programmers to use HME more conveniently. We extend the Glibc library to provide the nvm malloc function so that the application can allocate hybrid memories through malloc or nvm malloc. In order to implement the nvm malloc function, we modify Linux kernel to provide a branch for nvm mmap memory allocation. The branch handles the procedure alloc page in the nvm malloc function. Meanwhile, the NVM pages are differentiated from the DRAM pages in the original VMA (virtual address space). AHME marks the NVM flag in VMA, and calls the specified do nvm page fault function on a page fault to allocate the physical address space in the NUMA remote node. When nvm mmap() from the extended Glibc is called, the kernel calls the do mmap() and do mmap pgoff() functions and flags NVM VMA on the VMA structure when the do mmap pgoff() function applies for the VMA. When a NVM page is accessed at the first time, it generates a page fault and the kernel call handle mm fault() function to handle the page fault. If the NVM VMA flag is matched, the do nvm numa() function is used to allocate a physical page, which is allocated by alloc page() to DRAM on HME remote node (NVM). If the page fault do not refer to a NVM page, the kernel uses normal do page() to allocate DRAM. We implement nvm malloc function in Glibc library by referring to the malloc function, and it calls nvm mmap function through a new syscall. The nvm malloc() calls the nvm mmap() function to pass the mmap parameters to the kernel through MAP NVM parameter provided by AHME kernel. After that, the above NVM allocation is performedby the AHME kernel.

![Image of Yaktocat](https://github.com/Gumi-presentation-by-Dzh/HME/blob/master/images/memoryalloction.png)
![Image of Yaktocat](https://github.com/Gumi-presentation-by-Dzh/HME/blob/master/images/kernel.png)

AHME Setup,Compiling,Configuration and How to use
------------
**1.AHME kernel Compiling and install**

[![Build Status](https://travis-ci.org/RRZE-HPC/likwid.svg?branch=master)](https://travis-ci.org/RRZE-HPC/likwid)

* Compiling and Installation

From the emulator's source code /...._version/AHME/kernel,
```javascript
[root @node1 kernel]# cp config .config                            //To config the configration of linux kernel
[root @node1 kernel]# sh -c 'yes "" | make oldconfig'               //Use the old kernel configuration and automatically accept the default settings for each new option
[root @node1 kernel]# sudo make -j20 bzImage
[root @node1 kernel]# sudo make -j20 modules
[root @node1 kernel]# sudo make -j20 modules_install
[root @node1 kernel]# sudo make install
```
You can run 'sudo /...._version/AHME/kernel/bulid.sh' in order to automatically install

* Reboot to change to the AHME kernel
```javascript
[root @node1 kernel]# sudo vim /etc/default/grub      //To change the default kernel you using (always to set 0)
[root @node1 kernel]# sudo grub2-mkconfig -o /boot/grub2/grub.cfg
[root @node1 kernel]# reboot
```

**2.AHME Glibc Compiling and install**
* Compiling and Installation
From the emulator's source code /...._version/AHME/glibc,
```javascript
[root @node1 AHME]# mkdir glibc-build  
[root @node1 AHME]# cd glibc-build //to compiling the HME
[root @node1 glibc-build]# ../glibc/configure  --prefix=/usr --disable-profile --enable-add-ons --with-headers=/usr/include --with-binutils=/usr/bin
[root @node1 glibc-build]# make
[root @node1 glibc-build]# make install
```

**3.How to use AHME**
You can use this way to alloc nvm memory if you need.
```javascript
#include <malloc.h>
p = nvm_malloc(1024*8);
```
Then you must run this in type 0 (/HME/Regular_version/scripts/nvmini.in)

Happy hacking and hope you find SHMA useful for hybrid memory architecture research.

@Support or Contact

HME is developed in the HUST SCTS&CGCL Lab by ZhuoHui Duan, Haikun Liu and Xiaofei Liao. If you have any questions, please contact ZhuoHui Duan(zhduan@hust.edu.cn), Haikun Liu (hkliu@hust.edu.cn) and Xiaofei Liao (xfliao@hust.edu.cn). We welcome you to commit your modification to support our project.

## Support or Contact
This is developed in the [HUST SCTS&CGCL Lab](http://grid.hust.edu.cn/).
If you have any questions, please contact ZhuoHui Duan(zhduan@hust.edu.cn), Haikun Liu (hkliu@hust.edu.cn) and Xiaofei Liao (xfliao@hust.edu.cn).
We welcome you to commit your modification to support our project.