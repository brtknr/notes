# GPU Passthrough Tests

This document will try to cover both GPU passthrough device types:

* Intel integrated GPU
* nVidia GPU

For the Intel integrated GPU, the work will be based on this [guide](https://github.com/intel/gvt-linux/wiki/GVTg_Setup_Guide).

For PCI Passthrough in OpenStack:

* https://gist.github.com/claudiok/890ab6dfe76fa45b30081e58038a9215
* https://www.jimmdenton.com/gpu-offloading-openstack/
* https://docs.openstack.org/nova/latest/admin/pci-passthrough.html
* https://docs.openstack.org/nova/rocky/admin/pci-passthrough.html
* https://docs.openstack.org/nova/rocky/admin/virtual-gpu.html (vGPU only)

For Telsa models in OpenStack:

* https://egallen.com/openstack-nvidia-tesla-gpu-passthrough/

If it works, we may have to update this page with our nVidia card models:

* https://wiki.openstack.org/wiki/GPUs#GPU_passthrough_known_good.2Fworking_.26_bad.2Fnot-working_configs

## Must read

To understand a minimum what you'll do by following this document, you need at least to consult these links before going further.

* https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF
* https://www.kernel.org/doc/Documentation/Intel-IOMMU.txt
* https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.1/html/installation_guide/appe-configuring_a_hypervisor_host_for_pci_passthrough
* https://www.gresearch.co.uk/article/utilising-the-openstack-placement-service-to-schedule-gpu-and-nvme-workloads-alongside-general-purpose-instances/

## Instructions

### OpenStack - CentOS 7.x

> Without the contribution from Will, nothing would have been possible on the OpenStack part.

It is a little different than what I've seen on other systems.

1. Edit `dracut` module file
2. Run `dracut` command to regenerate the `initramfs` file
3. Edit the `grub2.conf` and `grub2-efi.conf` files
4. Blacklist the `nouveau` driver
5. Unbind the embedded USB controller with udev (won't be used on the guests anyway)
6. Bind the embedded USB controller to `vfio-pci`.

> For OpenStack only, continue with the steps below

7. Install the required __openstacksdk__ on the hypervisors (with GPU only)
8. Apply the __qemu__ wrapper

The use __Ansible__ file will be provided.

> __DON'T FORGET TO REMOVE THE `ifcfg-eth0` SCRIPT!!!__
>
> Otherwise the reboot will just fail...

This is what Will did on his side:

```bash
# Restart docker if disabled before proceed!
sudo systemctl restart docker

# Create the journalctl folder
sudo mkdir -vp /var/log/journal

# Patching the USB controller with UDEV rules
ACTION=="add", KERNEL=="0000:3b:00.2", SUBSYSTEM=="pci", RUN+="/bin/sh -c 'echo 0000\:3b\:00.2 > /sys/bus/pci/devices/0000\:3b\:00.2/driver/unbind'", RUN+="/bin/sh -c 'echo 10de 1ad8 > /sys/bus/pci/drivers/vfio-pci/new_id'"
```

> __The wrapper bellow will be replaced by the `hide_hypervisor_id` boolean option in the `gpu-computing` aggregate.__
>
> It can be normally also added in the flavour.

```bash
# Patching the licensing issue
gpu0# docker exec -it nova_libvirt bash
mv /usr/libexec/qemu-kvm /usr/libexec/qemu-kvm.orig && cat > /usr/libexec/qemu-kvm << EOF && chmod +x /usr/libexec/qemu-kvm
import os
import sys
new_args = []
for i in range(len(sys.argv)):
    if i<=1: 
        new_args.append(sys.argv[i])
        continue
    if sys.argv[i-1] != "-cpu":
        new_args.append(sys.argv[i])
        continue
    subargs = sys.argv[i].split(",")
    subargs.insert(1,"kvm=off")
    subargs.insert(2,"hv_vendor_id=MyFake_KVM")
    new_arg = ",".join(subargs)
    new_args.append(new_arg)
os.execv('/usr/libexec/qemu-kvm.orig', new_args)
EOF

# We also needed to restart nova_libvirt
gpu0# docker restart nova_libvirt
```

Regarding the nova scheduler, here is the main change:

```bash
# so at the moment any flavor that uses:
(venv) [will@jura-stackhpc ~]$ openstack aggregate show gpu-computing
+-------------------+----------------------------+
| Field             | Value                      |
+-------------------+----------------------------+
| availability_zone | None                       |
| created_at        | 2019-12-13T12:58:34.000000 |
| deleted           | False                      |
| deleted_at        | None                       |
| hosts             | [u'gpu0', u'gpu1']         |
| id                | 15                         |
| name              | gpu-computing              |
| properties        | hide_hypervisor_id='true'  |
| updated_at        | None                       |
+-------------------+----------------------------+
```

> The hide_hypervisor_id='true' will get scheduled to the gpu-computing aggregate

Here are the changes made on the OpenStack side:

* https://gitlab.isb-sib.ch/sib-openstack/sib-config/-/merge_requests/22/diffs
* https://gitlab.isb-sib.ch/sib-openstack/sib-kayobe-config/-/merge_requests/85/diffs

### Intel integrated GPU

> Not documented and partialy tested only. Will be added soon.

### nVidia GPU

#### Modify kernel boot options

This section will cover how to add the required boot options.

Add the following options with the method of your choice:

* `intel_iommu=on`
* `iommu=pt` (_Remove this option if you encounter issues with USB3 controller and devices_)
* `vfio-pci.ids=10de:13d7,10de:0fbb` (_I'll return back later on this one_)

##### Using GRUB config file

On Linux, [GRUB](https://en.wikipedia.org/wiki/GNU_GRUB) is generally used to manage kernel boot options (_and many other things_). Simply edit the configuration file to add new kernel boot options.

```bash
# Edit the GRUB config file
sudo nano /etc/default/grub
```

> Use any other text editor you like, no need to stick with `nano`.

And add new kernel boot options at the end of the line `GRUB_CMDLINE_LINUX` or `GRUB_CMDLINE_LINUX_DEFAULT` on some systems.

Result:

```bash
GRUB_CMDLINE_LINUX="nofb splash=quiet console=tty0 ... intel_iommu=on
```

> This is just an example, the command line might be different on your side.

##### Using Kernelstub

Some systems don't use [GRUB](https://en.wikipedia.org/wiki/GNU_GRUB) anymore to manage the kernel boot options, it's [Kernelstub](https://github.com/isantop/kernelstub) than is used instead.

```bash
# Print current config
sudo kernelstub -p

# Add kernel boot options
sudo kernelstub -a "options" -v -c

# Remove kernel boot options
sudo kernelstub -d "options" -v -c

# Show help
sudo kernelstub -h
```

> The __dry-run__ mode has been added to each commands (`-c`). Remove it to apply changes.

#### List IOMMU groups

It's something easy to do with this simple script:

```bash
#!/bin/bash
shopt -s nullglob
for g in /sys/kernel/iommu_groups/* ; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/* ; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done
done
```

[Source](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF)

Save it as `lsiommu.sh` and make it executable with `chmod -v +x lsiommu.sh`.

Sample output:

```
IOMMU Group 0:
    00:00.0 Host bridge [0600]: Intel Corporation Xeon E3-1200 v5/E3-1500 v5/6th Gen Core Processor Host Bridge/DRAM Registers [8086:1910] (rev 07)
IOMMU Group 1:
    00:01.0 PCI bridge [0604]: Intel Corporation Xeon E3-1200 v5/E3-1500 v5/6th Gen Core Processor PCIe Controller (x16) [8086:1901] (rev 07)
...
IOMMU Group 2:
    00:02.0 VGA compatible controller [0300]: Intel Corporation HD Graphics 530 [8086:191b] (rev 06)
...
```

> I've truncated the output and left only some interesting groups.

You can filter groups containing __VGA__ devices only with this command:

```bash
./lsiommu.sh | grep -i vga -B 1
```

Result:

```
IOMMU Group 2:
    00:02.0 VGA compatible controller [0300]: Intel Corporation HD Graphics 530 [8086:191b] (rev 06)
```

#### Display useful boot info

In order to understand and gather as more as possible the necessary informations to implement the GPU Passthrough, I've combined several commands to create a debug script.

```bash
#!/bin/bash

echo -e "\nVGA Arbiter:\n"
dmesg | grep -i vgaarb
sudo cat /dev/vga_arbiter | head -2

echo -e "\nIOMMU:\n"
dmesg | grep -i -e DMAR -e IOMMU

echo -e "\nDRM:\n"
dmesg | grep -i drm

echo -e "\nIntel GPU:\n"
dmesg | grep -i i915

echo -e "\nnVidia GPU:\n"
dmesg | grep -i nvidia

echo -e "\nVFIO:\n"
dmesg | grep -i vfio

echo -e "\nDetected display devices:\n"
sudo lspci -vnn | grep -i vga -A 10

echo -e "\nDetected nVidia devices:\n"
sudo lspci -vnn | grep -i nvidia -A 10

echo -e "\nDevices kernel modules:\n"
sudo lspci -knn | grep -i -e "hd graphics" -e "nvidia" -A 2

echo ""
```

Save it as `gpu-debug.sh` and make it executable with `chmod -v +x gpu-debug.sh`.

#### Load VFIO kernel modules

Before being able to pass the graphic card to the VMs, we need to load the __VFIO__ kernel modules.

```bash
# Edit the initramfs modules file
sudo nano /etc/initramfs-tools/modules
```

And add the following modules in this order:

```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

> The module `vfio_virqfd` might not be available for your kernel, remove it from the list if it fails.

Then save the file and your existing `initramfs`:

```bash
sudo update-initramfs -u
```

> To update all existing kernels, use `-uk all` instead of `-u`.

#### Attach PCI devices to VFIO

Now that the __VFIO__ kernel modules to load are defined, we can now attach the PCI device IDs to the __VFIO__ driver.

Collect the PCI device IDs:

```bash
# List all PCI devices and filter nVidia ones
lspci -vnn | grep -i nvidia -A 10
```

You should get something similar:

```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM204M [GeForce GTX 980M] [10de:13d7] (rev a1) (prog-if 00 [VGA controller])
	Subsystem: Acer Incorporated [ALI] GM204M [GeForce GTX 980M] [1025:105b]
	Flags: fast devsel
	Memory at 83000000 (32-bit, non-prefetchable) [disabled] [size=16M]
	Memory at 40000000 (64-bit, prefetchable) [disabled] [size=256M]
	Memory at 50000000 (64-bit, prefetchable) [disabled] [size=32M]
	I/O ports at a000 [disabled] [size=128]
	[virtual] Expansion ROM at 84000000 [disabled] [size=512K]
	Capabilities: <access denied>
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia

01:00.1 Audio device [0403]: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1)
	Subsystem: Acer Incorporated [ALI] GM204 High Definition Audio Controller [1025:0000]
	Flags: bus master, fast devsel, latency 0, IRQ 17
	Memory at 84080000 (32-bit, non-prefetchable) [size=16K]
	Capabilities: <access denied>
	Kernel driver in use: snd_hda_intel
	Kernel modules: snd_hda_intel
```

Here, __10de:13d7__ and __10de:0fbb__ are the PCI IDs we will have to pass to __VFIO__.

You can do a simple check without rebooting:

```bash
# Watch for hardware change
# You will see new lines appearing when PCI devices will be assigned
dmesg -wH

# On another terminal, load the VFIO kernel module and assign PCI devices
sudo modprobe vfio
sudo modprobe vfio_pci ids=10de:13d7,10de:0fbb

# List VFIO kernel modules
lsmod | grep vfio
```

You should have the following __VFIO__ kernel modules loaded:

```
vfio_pci               49152  0
vfio_virqfd            16384  1 vfio_pci
vfio_iommu_type1       28672  0
vfio                   32768  2 vfio_iommu_type1,vfio_pci
irqbypass              16384  5 vfio_pci,kvm
```

Now, run again this command to confirm that the nVidia devices are now using the __VFIO__ driver:

```bash
lspci -vnn | grep -i nvidia -A 10
```

If the `vfio_pci` driver is attached correctly to the device, then you can restart the `libvirtd` service to show the new available PCI devices:

```bash
# Restart the libvirtd service
sudo systemctl restart libvirtd.service ; systemctl status libvirtd.service
```

You should now see something like that in __Virt Manager__:

![image](./gpu-passthrough-pci-devices.png)

> You must attach all related PCI device from the same __IOMMU__ group, otherwise it will return as error when starting the VM.

#### Install nVidia Drivers on the VM

##### Windows guest VM

The driver should be installed by default once the new graphic card is detected.

Result when the VM is running:

![image](./win10-gpu-passthrough-okay-but-driver-issue.png)

> I'm still having to fix the famous error __43__ in order to make graphic working correctly on the Windows VM.
>
> According to this website: https://mathiashueber.com/fighting-error-43-nvidia-gpu-virtual-machine/, I might need to compile myself __qemu__ to the latest version to make it work correctly with Windows 10.

Some relevant information can be found here: https://github.com/qemu/qemu/blob/master/docs/hyperv.txt

##### Linux guest VM

In order to the make the passed GPU working in the VM, it is required to install the nVidia display / compute drivers.

```bash
sudo apt install nvidia-driver-440 nvidia-utils-440 nvidia-compute-utils-440
```

or for __headless__ install:

```bash
sudo apt install nvidia-headless-440 nvidia-utils-440 nvidia-compute-utils-440
```

You can also install `clinfo` to get information about the OpenCL driver installed.

```bash
sudo apt install clinfo
```

Result when the VM is running:

![image](./popos20.04-gpu-passthrough-okay.png)

> As you can see, it's much more easier on Linux guest VM :grin:

#### Run some GPU related commands on the VM

Here are some command results from the VM with the attached GPU.

##### Running `nvidia-smi`

```
Mon May 11 14:12:21 2020       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 440.82       Driver Version: 440.82       CUDA Version: 10.2     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 980M    Off  | 00000000:04:00.0 Off |                  N/A |
| N/A   60C    P8     7W /  N/A |      1MiB /  4043MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

##### Running `clinfo`

```
Number of platforms                               1
  Platform Name                                   NVIDIA CUDA
  Platform Vendor                                 NVIDIA Corporation
  Platform Version                                OpenCL 1.2 CUDA 10.2.159
  Platform Profile                                FULL_PROFILE
  Platform Extensions                             cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atomics cl_khr_fp64 cl_khr_byte_addressable_store cl_khr_icd cl_khr_gl_sharing cl_nv_compiler_options cl_nv_device_attribute_query cl_nv_pragma_unroll cl_nv_copy_opts cl_nv_create_buffer cl_khr_int64_base_atomics cl_khr_int64_extended_atomics
  Platform Extensions function suffix             NV

  Platform Name                                   NVIDIA CUDA
Number of devices                                 1
  Device Name                                     GeForce GTX 980M
  Device Vendor                                   NVIDIA Corporation
  Device Vendor ID                                0x10de
  Device Version                                  OpenCL 1.2 CUDA
  Driver Version                                  440.82
  Device OpenCL C Version                         OpenCL C 1.2 
  Device Type                                     GPU
  Device Topology (NV)                            PCI-E, 04:00.0
  Device Profile                                  FULL_PROFILE
  Device Available                                Yes
  Compiler Available                              Yes
  Linker Available                                Yes
  Max compute units                               12
  Max clock frequency                             1126MHz
  Compute Capability (NV)                         5.2
  Device Partition                                (core)
    Max number of sub-devices                     1
    Supported partition types                     None
    Supported affinity domains                    (n/a)
  Max work item dimensions                        3
  Max work item sizes                             1024x1024x64
  Max work group size                             1024
  Preferred work group size multiple              32
  Warp size (NV)                                  32
  Preferred / native vector sizes                 
    char                                                 1 / 1       
    short                                                1 / 1       
    int                                                  1 / 1       
    long                                                 1 / 1       
    half                                                 0 / 0        (n/a)
    float                                                1 / 1       
    double                                               1 / 1        (cl_khr_fp64)
  Half-precision Floating-point support           (n/a)
  Single-precision Floating-point support         (core)
    Denormals                                     Yes
    Infinity and NANs                             Yes
    Round to nearest                              Yes
    Round to zero                                 Yes
    Round to infinity                             Yes
    IEEE754-2008 fused multiply-add               Yes
    Support is emulated in software               No
    Correctly-rounded divide and sqrt operations  Yes
  Double-precision Floating-point support         (cl_khr_fp64)
    Denormals                                     Yes
    Infinity and NANs                             Yes
    Round to nearest                              Yes
    Round to zero                                 Yes
    Round to infinity                             Yes
    IEEE754-2008 fused multiply-add               Yes
    Support is emulated in software               No
  Address bits                                    64, Little-Endian
  Global memory size                              4240179200 (3.949GiB)
  Error Correction support                        No
  Max memory allocation                           1060044800 (1011MiB)
  Unified memory for Host and Device              No
  Integrated memory (NV)                          No
  Minimum alignment for any data type             128 bytes
  Alignment of base address                       4096 bits (512 bytes)
  Global Memory cache type                        Read/Write
  Global Memory cache size                        589824 (576KiB)
  Global Memory cache line size                   128 bytes
  Image support                                   Yes
    Max number of samplers per kernel             32
    Max size for 1D images from buffer            268435456 pixels
    Max 1D or 2D image array size                 2048 images
    Max 2D image size                             16384x16384 pixels
    Max 3D image size                             4096x4096x4096 pixels
    Max number of read image args                 256
    Max number of write image args                16
  Local memory type                               Local
  Local memory size                               49152 (48KiB)
  Registers per block (NV)                        65536
  Max number of constant args                     9
  Max constant buffer size                        65536 (64KiB)
  Max size of kernel argument                     4352 (4.25KiB)
  Queue properties                                
    Out-of-order execution                        Yes
    Profiling                                     Yes
  Prefer user sync for interop                    No
  Profiling timer resolution                      1000ns
  Execution capabilities                          
    Run OpenCL kernels                            Yes
    Run native kernels                            No
    Kernel execution timeout (NV)                 No
  Concurrent copy and kernel execution (NV)       Yes
    Number of async copy engines                  2
  printf() buffer size                            1048576 (1024KiB)
  Built-in kernels                                (n/a)
  Device Extensions                               cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atomics cl_khr_fp64 cl_khr_byte_addressable_store cl_khr_icd cl_khr_gl_sharing cl_nv_compiler_options cl_nv_device_attribute_query cl_nv_pragma_unroll cl_nv_copy_opts cl_nv_create_buffer cl_khr_int64_base_atomics cl_khr_int64_extended_atomics

NULL platform behavior
  clGetPlatformInfo(NULL, CL_PLATFORM_NAME, ...)  NVIDIA CUDA
  clGetDeviceIDs(NULL, CL_DEVICE_TYPE_ALL, ...)   Success [NV]
  clCreateContext(NULL, ...) [default]            Success [NV]
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_DEFAULT)  No platform
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_CPU)  No devices found in platform
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_GPU)  No platform
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_ACCELERATOR)  No devices found in platform
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_CUSTOM)  Invalid device type for platform
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_ALL)  No platform

ICD loader properties
  ICD loader Name                                 OpenCL ICD Loader
  ICD loader Vendor                               OCL Icd free software
  ICD loader Version                              2.2.11
  ICD loader Profile                              OpenCL 2.1
```

> __OpenCL__ will be used to run scientific applications. That's why `clinfo` is used in my tests.

## Performance tests

### Laptops with two graphic cards

I've made some performance tests on my own laptop. 

#### Tests with Hashcat

[Hashcat](https://hashcat.net/hashcat/) has been used to get a quick of the performances review between the Intel GPU and the nVidia GPU.

Simply run this command to install it:

```bash
sudo apt install hashcat
```

##### Intel GPU

```
$ hashcat -b -m0 --force -w3
hashcat (v4.0.1) starting in benchmark mode...

OpenCL Platform #1: Intel
=========================
* Device #1: Intel(R) HD Graphics Skylake Halo GT2, 3072/4096 MB allocatable, 24MCU

Benchmark relevant options:
===========================
* --force
* --workload-profile=3

Hashmode: 0 - MD5

Speed.Dev.#1.....:  3112.4 kH/s (57.74ms)

Started: Mon May 11 13:49:16 2020
Stopped: Mon May 11 13:49:21 2020
```

##### nVidia GPU (mobile)

```
$ hashcat -b -m0 -w3 --hwmon-disable
hashcat (v5.1.0) starting in benchmark mode...

OpenCL Platform #1: NVIDIA Corporation
======================================
* Device #1: GeForce GTX 980M, 1010/4043 MB allocatable, 12MCU

Benchmark relevant options:
===========================
* --workload-profile=3

Hashmode: 0 - MD5

Speed.#1.........:  3835.6 MH/s (52.03ms) @ Accel:512 Loops:128 Thr:256 Vec:1

Started: Mon May 11 13:46:47 2020
Stopped: Mon May 11 13:46:56 2020
```

As you can see the __nVidia GPU (mobile)__ is much more powerful than the __Intel GPU__.

* __Intel GPU__: 3112.4 kH/s (57.74ms)
* __nVidia GPU (mobile)__: 3835.6 __MH/s__ (52.03ms)

### OpenStack

I've made some performance tests on the newly created test VM. 

#### Tests with Hashcat

[Hashcat](https://hashcat.net/hashcat/) has been used to get a quick of the performances review between the Intel GPU and the nVidia GPU.

Simply run this command to install it:

```bash
sudo apt install hashcat
```

##### nVidia GPU

```
$ hashcat -b -m0 -w3
hashcat (v5.1.0) starting in benchmark mode...

OpenCL Platform #1: NVIDIA Corporation
======================================
* Device #1: GeForce RTX 2080, 1995/7982 MB allocatable, 46MCU

Benchmark relevant options:
===========================
* --workload-profile=3

Hashmode: 0 - MD5

Speed.#1.........: 16672.3 MH/s (45.87ms) @ Accel:256 Loops:256 Thr:256 Vec:1

Started: Wed May 13 10:21:44 2020
Stopped: Wed May 13 10:21:57 2020
```

As you can see the __nVidia GPU__ is much more powerful than my __Mobile nVidia GPU__.

* __nVidia GPU (mobile)__: 3835.6 MH/s (52.03ms)
* __nVidia GPU__: __16672.3__ MH/s (45.87ms)

## Research compilation

Here is just a quick summary about my observations:

* The default PCI device id for the second graphic card seems to be: `pci 0000:01:00.0`
* The onboard audio device of the graphic card is often: `pci 0000:01:00.1`
* The __IOMMU__ option `iommu=pt` does not seems to be well supported on my laptop
  * I had to remove it because my USB controller did not worked well anymore and I could not plug any USB3 devices.
  * Each time the USB3 device was plugged the Linux kernel has crashed.

### List available kernel settings

In order to get an idea of the available kernel settings that can be used on the kernel boot command line, you can run the following command:

```bash
$ ls /sys/module/kvm/parameters/
enable_vmware_backdoor  halt_poll_ns       halt_poll_ns_grow_start  ignore_msrs             lapic_timer_advance_ns  nx_huge_pages                 pi_inject_timer      tsc_tolerance_ppm
force_emulation_prefix  halt_poll_ns_grow  halt_poll_ns_shrink      kvmclock_periodic_sync  min_timer_period_us     nx_huge_pages_recovery_ratio  report_ignored_msrs  vector_hashing
```

You can also count them by running this command:

```bash
$ ls /sys/module/kvm/parameters/ | wc -l
16
```

### KVM kernel settings

> Occasionally, applications running in the VM may crash unexpectedly, whereas they would run normally on a physical machine. If, while running `dmesg -wH`, you encounter an error mentioning MSR, the reason for those crashes is that KVM injects a General protection fault (GPF) when the guest tries to access unsupported Model-specific registers (MSRs) - this often results in guest applications/OS crashing. A number of those issues can be solved by passing the ignore_msrs=1 option to the KVM module, which will ignore unimplemented MSRs.

[Source](https://wiki.archlinux.org/index.php/QEMU)

Sometimes the guest VM can hang and crash when using GPU Passthrough. The following kernel settings can fix the issue:

* `kvm.ignore_msrs=1`: Disable requests to MSRs but print a warning instead
* `kvm.report_ignored_msrs=0`: Do not print the warnings

### IOMMU kernel settings

According to the linux kernel source code (I've checked several versions, this part don't change actually), the IOMMU settings are the following:

* `intel_iommu=on`: IOMMU enabled
* `intel_iommu=off`: IOMMU disabled
* `iommu=pt`: Enable pass-through mode (Improve host performance)
* `iommu=igfx_off`: Disable GFX device mapping
* `iommu=forcedac`: Forcing DAC for PCI devices
* `iommu=strict`: Disable batched IOTLB flush
* `iommu=sp_off`: Disable supported super page
* `iommu=sm_on`: Intel-IOMMU: scalable mode supported
* `iommu=tboot_noforce`: Intel-IOMMU: not forcing on after tboot. This could expose security risk for tboot

> The setting `iommu=pt` can improve performance for systems using __DPDK__.
>
> See here for more details:
>
> * http://mails.dpdk.org/archives/dev/2014-October/007411.html
> * https://community.mellanox.com/s/article/understanding-the-iommu-linux-grub-file-configuration

Source code:

```c
// Near the begining of the file
static struct kset *iommu_group_kset;
static DEFINE_IDA(iommu_group_ida);
#ifdef CONFIG_IOMMU_DEFAULT_PASSTHROUGH
static unsigned int iommu_def_domain_type = IOMMU_DOMAIN_IDENTITY;
#else
static unsigned int iommu_def_domain_type = IOMMU_DOMAIN_DMA;
#endif
static bool iommu_dma_strict __read_mostly = true;

// Later in the file
static int __init iommu_set_def_domain_type(char *str)
{
    bool pt;
    int ret;

    ret = kstrtobool(str, &pt);
    if (ret)
        return ret;

    iommu_def_domain_type = pt ? IOMMU_DOMAIN_IDENTITY : IOMMU_DOMAIN_DMA;
    return 0;
}
early_param("iommu.passthrough", iommu_set_def_domain_type);

static int __init iommu_dma_setup(char *str)
{
    return kstrtobool(str, &iommu_dma_strict);
}
early_param("iommu.strict", iommu_dma_setup);
```

[Source](https://elixir.bootlin.com/linux/v5.3.18/source/drivers/iommu/iommu.c)

```c
if (!strncmp(str, "on", 2)) {
    dmar_disabled = 0;
    pr_info("IOMMU enabled\n");
} else if (!strncmp(str, "off", 3)) {
    dmar_disabled = 1;
    no_platform_optin = 1;
    pr_info("IOMMU disabled\n");
} else if (!strncmp(str, "igfx_off", 8)) {
    dmar_map_gfx = 0;
    pr_info("Disable GFX device mapping\n");
} else if (!strncmp(str, "forcedac", 8)) {
    pr_info("Forcing DAC for PCI devices\n");
    dmar_forcedac = 1;
} else if (!strncmp(str, "strict", 6)) {
    pr_info("Disable batched IOTLB flush\n");
    intel_iommu_strict = 1;
} else if (!strncmp(str, "sp_off", 6)) {
    pr_info("Disable supported super page\n");
    intel_iommu_superpage = 0;
} else if (!strncmp(str, "sm_on", 5)) {
    pr_info("Intel-IOMMU: scalable mode supported\n");
    intel_iommu_sm = 1;
} else if (!strncmp(str, "tboot_noforce", 13)) {
    printk(KERN_INFO
        "Intel-IOMMU: not forcing on after tboot. This could expose security risk for tboot\n");
    intel_iommu_tboot_noforce = 1;
}
```

[Source](https://elixir.bootlin.com/linux/v5.3.18/source/drivers/iommu/intel-iommu.c)

### Custom bind script (do not use, kept here for later testing)

I'll certainly use this script and I may change it / fix it according to my needs.

```bash
#!/bin/bash
#Comment
#More Echos
#Fix logout crash
#Add exceptions to on/off to avoid freezing

usage() {
    echo -e "\nBind / Unbind discrete or any graphic cards"
    echo -e "\n  !! Edit the script to change the PCI IDs before running the script !!\n"
    echo -e "How: Run lspci with the card(s) enabled to get them.\n"
    echo -e "Usage: $0 <action/bind-device-type> (vfio, nouveau, nvidia, unbind, on, off)\n"
}

vfio() {
    if ! $(lspci -H1 | grep --quiet NVIDIA)
    then
        on
        sleep 1
    fi
    unbind
    echo 0x10de 0x1ba1 > /sys/bus/pci/drivers/vfio-pci/new_id
    echo 0x10de 0x10f0 > /sys/bus/pci/drivers/vfio-pci/new_id
    echo 1 > /sys/bus/pci/rescan
}
 
nouveau() {
    if ! $(lspci -H1 | grep --quiet NVIDIA)
    then
        on
        sleep 1
    fi
    unbind
    echo 0x10de 0x1ba1 > /sys/bus/pci/drivers/nouveau/new_id
    echo "0000:01:00.1" > /sys/bus/pci/drivers/snd_hda_intel/bind
    echo 1 > /sys/bus/pci/rescan
}
 
nvidia() {
    if ! $(lspci -H1 | grep --quiet NVIDIA)
    then
        on
        sleep 1
    fi
    unbind
    echo 0x10de 0x1ba1 > /sys/bus/pci/drivers/nvidia/new_id
    echo "0000:01:00.1" > /sys/bus/pci/drivers/snd_hda_intel/bind
    echo 1 > /sys/bus/pci/rescan
}
 
unbind() {
    if [ -f /sys/bus/pci/devices/0000\:01\:00.0/driver/unbind ]
    then
        echo "0000:01:00.0" > /sys/bus/pci/devices/0000\:01\:00.0/driver/unbind
    fi
    if [ -f /sys/bus/pci/devices/0000\:01\:00.1/driver/unbind ]
    then
        echo "0000:01:00.1" > /sys/bus/pci/devices/0000\:01\:00.1/driver/unbind
    fi
}
 
on() {
    echo '\_SB_.PCI0.PEG0.PEGP._ON' > /proc/acpi/call
    if [ -f /sys/bus/pci/devices/0000\:01\:00.0/remove ]
    then
        echo 1 > /sys/bus/pci/devices/0000\:01\:00.0/remove
    fi
    if [ -f /sys/bus/pci/devices/0000\:01\:00.1/remove ]
    then
        echo 1 > /sys/bus/pci/devices/0000\:01\:00.1/remove
    fi
    echo 1 > /sys/bus/pci/rescan
}
 
off() {
    unbind
    echo '\_SB_.PCI0.PEG0.PEGP._OFF' > /proc/acpi/call
}
 
case "$1" in
    boot)
        boot
        ;;
    vfio)
        vfio
        ;;
    nouveau)
        nouveau
        ;;
    nvidia)
        nvidia
        ;;
    unbind)
        unbind
        ;;
    on)
        on
        ;;
    off)
        off
        ;;
    *)
        usage
        ;;
esac
```

[Source](https://gist.github.com/Misairu-G/616f7b2756c488148b7309addc940b28#gistcomment-3257565)

### Useful commands

```bash
# Debug VGA
dmesg | grep -i vgaarb
sudo cat /dev/vga_arbiter | head -2

# Debug VFIO
dmesg | grep -i vfio

# Debug DRM
dmesg | grep -i drm

# Debug IOMMU
dmesg | grep -i iommu

# Full IOMMU Debug
dmesg | grep -i -e DMAR -e IOMMU

# Debug i915 chipset (Intel Integrated GPU)
dmesg | grep -i i915

# Debug nVidia
dmesg | grep -i nvidia

# Debug GLX
glxinfo | head

# List all PCI devices
lspci -vnn

# List video devices
lspci -vnn | grep -i vga -A 10

# List loaded device kernel drivers per device
lspci -nnk

# Show CPU pinning topology
lscpu -e
```

## Troubleshooting

### Driver not working in the VM

The latest nVidia driver is checking if the card is passed into a virtual machine then stop working it that's the case. To workaround this check, we can hide the presence of __KVM__.

#### Linux guest VMs

Symptoms:

```
DRM:

[    3.176628] [drm] [nvidia-drm] [GPU ID 0x00000005] Loading driver
[    3.180389] [drm] Initialized nvidia-drm 0.0.0 20160202 for 0000:00:05.0 on minor 0
[    6.325467] systemd[1]: Condition check resulted in Load Kernel Module drm being skipped.

Intel GPU:
                                                                                                         

nVidia GPU:

[    2.955660] nvidia: loading out-of-tree module taints kernel.
[    2.968185] nvidia: module license 'NVIDIA' taints kernel.
[    2.989003] nvidia: module verification failed: signature and/or required key missing - tainting kernel
```

> The script `gpu-debug.sh` has been used to collect the information above.

Conclusion:

__It has failed for the same reason like it does on Windows... Licensing issue.__

> Fortunately, a solution exist and work on linux guest VMs. See below.

Before patch:

```
ubuntu@willszumski-test:~$ nvidia-smi 
Unable to determine the device handle for GPU 0000:00:05.0: Unknown Error
```

After patch:

```
ubuntu@willszumski-test:~$ nvidia-smi                                                                     
Tue May 12 16:46:01 2020       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 440.64       Driver Version: 440.64       CUDA Version: 10.2     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce RTX 2080    Off  | 00000000:00:05.0 Off |                  N/A |
| 24%   37C    P0    20W / 225W |      0MiB /  7982MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

Follow the instructions from these links to apply the patch:

* https://bugs.launchpad.net/nova/+bug/1752463
* https://gist.github.com/claudiok/890ab6dfe76fa45b30081e58038a9215

#### Windows guest VMs

The famous error 43 continue to beat me again and again... I've tried almost everything possible a part compile `qemu` which I think is the last solution to try.

Here is how the error look like:

![image](./win10-gpu-passthrough-okay-but-driver-issue.png)

### VM is crashing for no reasons

Try to apply the following patch to the hypervisor:

```bash
# For the current boot
sudo -s
echo 1 > /sys/module/kvm/parameters/ignore_msrs

# To make it permanent after reboot
sudo nano /etc/modprobe.d/kvm.conf
options kvm ignore_msrs=Y
options kvm report_ignored_msrs=N
```

## Credits

* Jonathan Barda - SIB (research on GPU / PCI passthrough, tests, implementation, documentation)
* Volker Flegel - SIB (research on PCI passthrough)
* Will Szumsky - StackHPC (brainstorming, deployment on OpenStack)

## References

* https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF
* https://wiki.archlinux.org/index.php/QEMU
* https://wiki.archlinux.org/index.php/NVIDIA_Optimus
* https://scottlinux.com/2016/08/28/gpu-passthrough-with-kvm-and-debian-linux/
* https://davidyat.es/2016/09/08/gpu-passthrough/
* https://taxes.dev/2017/07/08/linux-and-windows-running-simultaneously-with-gpu-passthrough/
* http://lists.openstack.org/pipermail/openstack-operators/2018-March/014988.html
* https://gist.github.com/Misairu-G/616f7b2756c488148b7309addc940b28
* https://gist.github.com/Misairu-G/616f7b2756c488148b7309addc940b28#gistcomment-3257565
* https://mathiashueber.com/windows-virtual-machine-gpu-passthrough-ubuntu/
* https://bufferoverflow.io/gpu-passthrough/
* https://github.com/noizwaves/vfio-popos-nvidia-ga-z370-gaming-5
* https://www.reddit.com/r/VFIO/comments/atp689/gpu_passthrough_on_popos_1804_need_help_with_vfio/
* https://www.reddit.com/r/VFIO/comments/fakeqc/gpu_passthrough_tutorial_pop_ossystemd_distros/
* https://github.com/bryansteiner/gpu-passthrough-tutorial
* https://forum.level1techs.com/t/gpu-passthrough-on-popos-18-04-need-help-with-vfio-driver-override/139270
* https://forum.level1techs.com/t/gpu-passthrough-on-popos-using-pcie-id-instead-of-manufacturer-id/142871/3
* https://forum.level1techs.com/t/vfio-in-2019-pop-os-how-to-general-guide-though-draft/142287?u=michaellindman
* https://forum.level1techs.com/t/play-games-in-windows-on-linux-pci-passthrough-quick-guide/108981
* https://www.redhat.com/archives/vfio-users/2015-September/msg00471.html
* https://elixir.bootlin.com/linux/v5.3.18/source/drivers/iommu/intel-iommu.c
* https://elixir.bootlin.com/linux/v5.3.18/source/drivers/iommu/iommu.c
* https://dpdk-guide.gitlab.io/dpdk-guide/setup/iommu.html
* https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.1/html/installation_guide/appe-configuring_a_hypervisor_host_for_pci_passthrough
* https://wiki.ubuntu.com/Bumblebee
* https://askubuntu.com/questions/1031511/cant-disable-nouveau-drivers-in-ubuntu-18-04
* https://askubuntu.com/questions/1029169/bumblebee-doesnt-work-on-ubuntu-18-04/1042950#1042950
* https://docs.openstack.org/nova/latest/admin/pci-passthrough.html
* https://clayfreeman.github.io/gpu-passthrough/
* https://www.reddit.com/r/VFIO/comments/bjbes0/how_to_force_usb_30_controller_to_bind_to_vfio/
* https://www.reddit.com/r/VFIO/comments/ergu7m/windows_10_crashes_on_login/
* https://patchwork.kernel.org/patch/10048427/
* https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/14/html/instances_and_images_guide/ch-virtual-gpu