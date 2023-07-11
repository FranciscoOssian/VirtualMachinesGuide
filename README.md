# VirtualMachinesGuide

Backup of my journey learning about virtualization with Virt-Manager and Libvirt. Shared for anyone interested in these discoveries.

# Guide to Virtual Machines, Virt Manager, QEMU, and KVM

This repository is a guide to understanding and using virtual machines, with special focus on Virt Manager, which uses QEMU and KVM. The goal is to document my experiences and discoveries in this field and to assist others who may be interested in these technologies.

I had two main sources of information on YouTube, a quicker one and a more detailed one:

1. [Games in Virtual Machine with GPU Passthrough | Understanding QEMU, KVM, Libvirt (pt-br)](https://www.youtube.com/watch?v=IDnabc3DjYY&ab_channel=FabioAkita)
2. [Windows 11 on KVM – How to Install Step by Step (en)](https://getlabsdone.com/how-to-install-windows-11-on-kvm/)

## Table of Contents

1. [Checking virtualization support](#checking-virtualization-support)
2. [Installing Virt-Manager](#installing-virt-manager)
3. [Downloading the desired ISO](#downloading-the-desired-iso)
4. [Creating your first virtual machine](#creating-your-first-virtual-machine)
5. [Optimizations](#optimizations)
6. [Problems and solutions](#problems-and-solutions)
7. [Guess is Win11](#guess-is-win11)
8. [Scripts and code samples](#scripts-and-code-samples)

## Checking virtualization support

The first step before installing QEMU and KVM is to check if your machine supports virtualization.

_instructions._

## Installing Virt-Manager

After verifying virtualization support, the next step is to install Virt-Manager, a user-friendly graphical interface for managing virtual machines.

_Note: Installing Virt-Manager typically also includes installing QEMU and KVM, but depending on your distribution, it may be necessary to install them separately. This guide does not cover everything about these tools, just the main parts. Remember, this is primarily a personal backup of the things I will need._

Here are the steps to install Virt-Manager...

[https://virt-manager.org/](https://virt-manager.org/)
This is the homepage, which already shows a section on how to install. I won't go into much detail here about the installation.
Just out of curiosity, I am currently testing on a Debian x86_64.

## Downloading the desired ISO

After installing Virt-Manager (and consequently QEMU and KVM), you should download the ISO of the operating system you want to install on your virtual machine.

This section is important if you want to use the VirtIO standard, which is used exclusively in virtual machines.

For Linux VMs compatibility is more build-in, but if you want to emulate Windows and use VirtIO, you should get the .iso (drivers) for the devices. Resuming, VirtIO is a standard like SATA, but optimized for VMs.

When we use VirtIO to create VMs, such as Windows, we will in some cases need to configure VirtIO drivers when creating the machine and after install. I will go into this in more detail in a separate topic.

## Creating your first virtual machine

With Virt-Manager installed and the ISO downloaded, you're ready to create your first virtual machine!

- In Virt-Manager, click on create a new VM.
- Choose the ISO option if you wish (which is the one covered here).
- Proceed and choose the ISO you want. Let Virt-Manager identify the system type of this ISO or not.
- Choose the amount of RAM and CPU that the VM will have, as well as disk capacity.
- I recommend choosing to customize the machine configurations before starting, especially a non-Linux system (we still need to configure VirtIO, TPM 2.0 if necessary).

I always leave the Network option on NAT, as it covers all my cases. I don't plan on covering the other examples here, as I don't see use cases for my work.

## Optimizations

### Bottlenecks, running a VM on a host with a 4-core processor

Without any of the listed optimizations, heavier VMs such as Windows 11 will not work properly on my computer. I'm using a quad-core Intel i5-8250U processor.

Here's the topology of my processor:
```shell
➜  ~ lscpu -e
CPU NODE SOCKET CORE L1d:L1i:L2:L3 ONLINE    MAXMHZ   MINMHZ       MHZ
  0    0      0    0 0:0:0:0          yes 1600,0000 400,0000  624,9860
  1    0      0    1 1:1:1:0          yes 1600,0000 400,0000 1800,0000
  2    0      0    2 2:2:2:0          yes 1600,0000 400,0000  680,2230
  3    0      0    3 3:3:3:0          yes 1600,0000 400,0000 1800,0000
  4    0      0    0 0:0:0:0          yes 1600,0000 400,0000  657,9750
  5    0      0    1 1:1:1:0          yes 1600,0000 400,0000  613,4460
  6    0      0    2 2:2:2:0          yes 1600,0000 400,0000 1800,0000
  7    0      0    3 3:3:3:0          yes 1600,0000 400,0000 1800,0000
➜  ~
```

Here we can see that my processor has 4 cores, each with 2 threads (Concurrent Multi-threading).

You can see on your computer, which is easier to understand using `lstopo` - a command from the `hwloc` package. It displays this information graphically.

```shell
➜  ~ lstopo-no-graphics     
Machine (xGB total)
  Package L#0
    NUMANode L#0 (P#0 xGB)
    L3 L#0 (6144KB)
      L2 L#0 (256KB) + L1d L#0 (32KB) + L1i L#0 (32KB) + Core L#0
        PU L#0 (P#0)
        PU L#1 (P#4)
      L2 L#1 (256KB) + L1d L#1 (32KB) + L1i L#1 (32KB) + Core L#1
        PU L#2 (P#1)
        PU L#3 (P#5)
      L2 L#2 (256KB) + L1d L#2 (32KB) + L1i L#2 (32KB) + Core L#2
        PU L#4 (P#2)
        PU L#5 (P#6)
      L2 L#3 (256KB) + L1d L#3 (32KB) + L1i L#3 (32KB) + Core L#3
        PU L#6 (P#3)
        PU L#7 (P#7)
```

### Processor Partitioning

#### Methods

There are two complementary methods that I've discovered, you can use one or both together.

1. [L3 Cache Optimization](#l3-cache-optimization)
2. [Simultaneous Multi-threading (SMT) Optimization](#simultaneous-multi-threading-smt-optimization)

These methods optimize VM performance by minimizing resource contention between the host and the guest. The SMT optimization in particular is beneficial for processors that divide a single core into two or more threads, essentially virtualizing more CPUs to the operating system.

#### L3 Cache Optimization

L3 is a slower cache but has more space and is shared across multiple cores. To streamline OS tasks, it's good for your system if the VM and Host share as many cores as possible under the same L3 cache. You can identify these scopes by running `lscpu -e` or `lstopo`.

#### Simultaneous Multi-threading (SMT) Optimization

Some processors have more than one thread within a single core. For better optimization, it's best to ensure that host and guest don't share threads from the same core.

#### Results

After applying both optimizations, the assignment is as follows (for my Host and Guess):

```shell
Host = { 0,4,1,5 }
Guest = { 2,6,3,7 }
```

The shell script I've developed uses systemd and libvirt hooks to manage this allocation. It restricts the host system to use the defined CPUs on VM start. On VM shutdown, if no other VMs are active, it allows the host system to use all CPUs. If other VMs are still running, it maintains the current CPU allocation.

I've also configured the CPU topology and affinity for each vCPU of the VM in the VM's settings. This informs libvirt that the emulated processor will have 4 cores, which are set to use only the allocated cores.

```xml
  <vcpu placement="static">4</vcpu>
  <cputune>
    <vcpupin vcpu="0" cpuset="2"/>
    <vcpupin vcpu="1" cpuset="6"/>
    <vcpupin vcpu="2" cpuset="3"/>
    <vcpupin vcpu="3" cpuset="7"/>
    <emulatorpin cpuset="0-1,4-5"/>
  </cputune>
```

Finally, Virt Manager is aware of the virtual processor topology.

```xml
  <cpu ...args>
    <topology sockets="1" dies="1" cores="2" threads="2"/>
  </cpu>
```

## Problems and solutions

The only problem (with no solution) that I had is the screen proportion on win11.

## Guess is Win11

### TPM 2.0

Windows 11 requires TPM 2.0, which some computers lack. Therefore, if your host machine does not have TPM 2.0, you'll need to configure an emulated TPM 2.0. If your host machine does have TPM 2.0, you can use passthrough.

Here's my configuration for a host machine without TPM 2.0:

```xml
<tpm model="tpm-tis">
  <backend type="emulator" version="2.0"/>
</tpm>
```

Here's an example configuration with passthrough:

```xml
<tpm model="tpm-tis">
  <backend type="passthrough">
    <device path="/dev/tpm0"/>
  </backend>
</tpm>
```

### VirtIO?

I use virtIO in:

- VM Disk
- network interface
- Video interface

```xml
<disk type="file" device="disk">
  <driver name="qemu" type="qcow2" discard="unmap"/>
  <source file="/var/lib/libvirt/images/win11-example.qcow2"/>
  <target dev="vda" bus="virtio"/>
  <boot order="1"/>
  <address type="pci" domain="0x0000" bus="0x04" slot="0x00" function="0x0"/>
</disk>
```

```xml
<interface type="network">
  <mac address="xxx"/>
  <source network="default"/>
  <model type="virtio"/>
  <address type="pci" domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
</interface>
```

```xml
<graphics type="spice">
  <listen type="none"/>
  <image compression="off"/>
  <gl enable="yes" rendernode="/dev/dri/by-path/pci-0000:00:02.0-render"/>
</graphics>
```

```xml
<video>
  <model type="virtio" heads="1" primary="yes">
    <acceleration accel3d="yes"/>
  </model>
  <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0"/>
</video>
```

You will need install this drivers after install if emulating Win.

If you use virtIO in disk, will need install the driver for disk when installing Win.

## Scripts and code samples

➜ ~ cat /etc/libvirt/hooks/qemu

script not complete, because when shutting down the VM it is not working, just when start.

```shell
#!/bin/bash

# Function to check if any VM is running
one_or_more_running() {
    vms=$(sudo virsh list --state-running --name | tr -d '\n')
    if [ -z "$vms" ]; then
        return 1
    else
        return 0
    fi
}

command=$2

# When starting the VM
if [ "$command" = "start" ]; then
    # Disables cores for the host system
    # Your command here
    sudo systemctl set-property --runtime -- system.slice AllowedCPUs=0,4,1,5
    sudo systemctl set-property --runtime -- user.slice AllowedCPUs=0,4,1,5
    sudo systemctl set-property --runtime -- init.scope AllowedCPUs=0,4,1,5
fi

# When shutting down the VM
if [ "$command" = "shutdown" ]; then
    # Checks if any VM is still running
    # Wait for a few seconds to give the VM time to shut down
    sleep 6
    if one_or_more_running; then
        echo "One or more VMs are still running"
    else
	echo "reset"
        # If not, enables cores for the host system
        # Your command here
        sudo systemctl set-property --runtime -- system.slice AllowedCPUs=0-7
        sudo systemctl set-property --runtime -- user.slice AllowedCPUs=0-7
        sudo systemctl set-property --runtime -- init.scope AllowedCPUs=0-7
    fi
fi
```
