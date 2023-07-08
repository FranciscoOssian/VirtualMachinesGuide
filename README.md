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
7. [Running Win11](#running-win11)
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

## bottlenecks, running a VM on host with 8-core processor

I use a script to allocate half of the cores exclusively to the VM upon startup, leveraging systemd and libvirt's hooks. If only one VM is running, it utilizes half of the CPU power.

**Pseudocode:**

1. On VM start: Restrict host system to use only half of the CPUs (0 to N).
2. On VM shutdown: If no other VMs are active, allow host system to use all CPUs. If other VMs are still running, maintain the current CPU allocation.

More details can be found in the 'Scripts and code samples' section.

In the CPU topology of the VM, I've created an XML CPU pin configuration to map the CPU, telling libvirt that the emulated processor will have 4 cores, which are set to use only the chosen cores.

```xml
  <vcpu placement="static">4</vcpu>
  <cputune>
    <vcpupin vcpu="0" cpuset="4"/>
    <vcpupin vcpu="1" cpuset="5"/>
    <vcpupin vcpu="2" cpuset="6"/>
    <vcpupin vcpu="3" cpuset="7"/>
    <emulatorpin cpuset="0-3"/>
  </cputune>
```

This tell to to libvirt that the emulated processor will have 4 cores, pointed to use just the choosed cores.

To know what is the best range to use in VMs. Choose a range that have the same L3 cache. In my case is any.

Make sure that Virt Manager know about the virtual processor topology. 
```xml
  <cpu ...args>
    <topology sockets="1" dies="1" cores="4" threads="1"/>
  </cpu>
```
4 cores, defined in `cputune`


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

## Problems and solutions

The only problem (with no solution) that I had is the screen proportion on win11.

## Running Win11

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

```shell
#!/bin/bash

# Function to check if any VM is running
check_vms() {
    vms=$(virsh list --all | grep running)
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
    systemctl set-property --runtime -- system.slice AllowedCPUs=0,1,2,3
    systemctl set-property --runtime -- user.slice AllowedCPUs=0,1,2,3
    systemctl set-property --runtime -- init.scope AllowedCPUs=0,1,2,3
fi

# When shutting down the VM
if [ "$command" = "shutdown" ]; then
    # Checks if any VM is still running
    if check_vms; then
        echo "One or more VMs are still running"
    else
        # If not, enables cores for the host system
        # Your command here
        systemctl set-property --runtime -- system.slice AllowedCPUs=0-7
        systemctl set-property --runtime -- user.slice AllowedCPUs=0-7
        systemctl set-property --runtime -- init.scope AllowedCPUs=0-7
    fi
fi
```
