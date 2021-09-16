Most of the time I have followed this procedure: https://www.reddit.com/r/VFIO/comments/npr2uo/successful_single_gpu_passthrough/ 

Here I shall document where I deviated from this. 

**System Information**
````
OS: MX x86_64 
Host: Z87P-D3 
Kernel: 5.10.0-5mx-amd64 
Uptime: 1 hour, 20 mins 
Packages: 3001 (dpkg), 20 (brew), 34 (flatpak) 
Shell: bash 5.0.3 
Resolution: 2560x1440, 2560x1440, 1920x1080 
DE: Plasma 5.14.5 
WM: KWin 
Theme: Breeze Dark [Plasma], Breeze-Dark [GTK2/3] 
Icons: breeze-dark [Plasma], breeze-dark [GTK2/3] 
Terminal: yakuake 
Terminal Font: Andale Mono 14 
CPU: Intel i7-4770 (8) @ 3.900GHz 
GPU: NVIDIA GeForce GTX 1070 
Memory: 4564MiB / 32087MiB 

````


## Step 1: enable IOMMU

In MX linux you can edit grub via the tool 'MX Boot Options', under 'Kernel parameters'. For my intel CPU, inserting `intel_iommu=on iommu=pt` sufficed. 

For the iommu groups the most important thing is that your gpu and audio device of your graphics card are in the same group or in a separate group. Having a pcie controller in there as well is fine, as this is the controller your card is connected with. In my case the group looked like this:

````
IOMMU Group 1:
        00:01.0 PCI bridge [0604]: Intel Corporation Xeon E3-1200 v3/4th Gen Core Processor PCI Express x16 Controller [8086:0c01] (rev 06)
        01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP104 [GeForce GTX 1070] [10de:1b81] (rev a1)
        01:00.1 Audio device [0403]: NVIDIA Corporation GP104 High Definition Audio Controller [10de:10f0] (rev a1)
````
## Step 2: install packages

Packages relevant to MX linux are `qemu qemu-kvm libvirt  virt-manager and dnsmasq`. Virt-manager I got from the debian backports, as this is the only version (1:2.2.1-3-bpo10+1) that allows editing the XML file. Remember that by default this is disabled, so in virt-manager go to preferences->general->enable xml editing.

## Step 3: VM preparation

In order to be able to set firmware to OVMF, you need to install the ovmf package. 

With regards to CPU, I had to set topology for my quad core to:

````
Sockets: 1
Cores: 4
Threads: 2
````
With ``virsh capabilities | grep topology``  you can find out the topology of your cpu. 

## Step 4: Prepare the directory structure

My kvm.conf:
````
# /etc/libvirt/hooks/kvm.conf
VIRSH_GPU_VIDEO=pci_0000_01_00_0
VIRSH_GPU_AUDIO=pci_0000_01_00_1

````
My start.sh:

````
#!/bin/bash
 
# set debugging
set -x
 
# load variables
source "/etc/libvirt/hooks/kvm.conf"
 
# stop display manager
systemctl stop sddm.service
 
# unbind VTConsoles
echo 0 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind
 
# unbind efi framebuffer
echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind
 
# avoid race condition
sleep 2
 
#unload nvidia
modprobe -r nvidia_uvm
modprobe -r nvidia_drm
modprobe -r nvidia_modeset
modprobe -r drm_kms_helper
modprobe -r drm
modprobe -r nvidia
 
# unbind the GPU
virsh nodedev-detach $VIRSH_GPU_VIDEO
virsh nodedev-detach $VIRSH_GPU_AUDIO
 
# load vfio
modprobe vfio
modprobe vfio_pci
modprobe vfio_iommu_type1
````
My revert.sh:

````
#!/bin/bash
 
# enable debugging
set -x
 
# load vars
source "/etc/libvirt/hooks/kvm.conf"


# unload vfio pci
modprobe -r vfio_iommu_type1
modprobe -r vfio_pci
modprobe -r vfio
 
# rebind GPU
virsh nodedev-reattach $VIRSH_GPU_VIDEO
virsh nodedev-reattach $VIRSH_GPU_AUDIO
 
# rebind VTConsoles
echo 1 > /sys/class/vtconsole/vtcon0/bind
echo 1 > /sys/class/vtconsole/vtcon1/bind
 
# revive nvidia
nvidia-xconfig --query-gpu-info > /dev/null 2>&1
 
# bind EFI-framebuffer
echo "efi-framebuffer.0" > /sys/bus/platform/drivers/efi-framebuffer/bind
 
# load nvidia
# nvidia_uvm
modprobe nvidia_drm
modprobe nvidia_modeset
modprobe drm_kms_helper
modprobe drm
modprobe nvidia
 
systemctl start sddm.service
````

## Step 5: GPU Jacking

In my case most of my USB devices were in IOMMU group 2:
````
Bus 2 --> 0000:00:14.0 (IOMMU group 2)
Bus 002 Device 008: ID 1b1c:1b50 Corsair 
Bus 002 Device 010: ID 0a12:0001 Cambridge Silicon Radio, Ltd Bluetooth Dongle (HCI mode)
Bus 002 Device 009: ID 062a:4c01 MosArt Semiconductor Corp. 
Bus 002 Device 006: ID 0bda:5411 Realtek Semiconductor Corp. 
Bus 002 Device 004: ID 0bda:5411 Realtek Semiconductor Corp. 
Bus 002 Device 003: ID 0482:06b1 Kyocera Corp. 
Bus 002 Device 002: ID 045e:0719 Microsoft Corp. Xbox 360 Wireless Adapter
Bus 002 Device 005: ID 1058:1021 Western Digital Technologies, Inc. Elements Desktop (WDBAAU)
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

````

## Step 6: XML editing

For me, I had to perform an extra step. Due to Nvidia shenanigans I had to add the VBIOS ROM manually to my VM. You can find these on [Techpowerup](https://www.techpowerup.com/vgabios/). Afterwards I had to patch it with the [NVIDIA VFIO patcher](https://github.com/Matoking/NVIDIA-vBIOS-VFIO-Patcher). This is only relevant for Nvidia 1xxx (Pascal series) cards.

My final XML:
````XML
<domain type="kvm">
  <name>win10</name>
  <uuid>27e3fd44-7cce-441d-be2a-bc0f807b6d94</uuid>
  <metadata>
    <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
      <libosinfo:os id="http://microsoft.com/win/10"/>
    </libosinfo:libosinfo>
  </metadata>
  <memory unit="KiB">8388608</memory>
  <currentMemory unit="KiB">8388608</currentMemory>
  <vcpu placement="static">8</vcpu>
  <os>
    <type arch="x86_64" machine="pc-q35-3.1">hvm</type>
    <loader readonly="yes" type="pflash">/usr/share/OVMF/OVMF_CODE.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/win10_VARS.fd</nvram>
    <boot dev="hd"/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <hyperv>
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vpindex state="on"/>
      <runtime state="on"/>
      <synic state="on"/>
      <stimer state="on"/>
      <reset state="on"/>
      <vendor_id state="on" value="sashka"/>
      <frequencies state="on"/>
    </hyperv>
    <kvm>
      <hidden state="on"/>
    </kvm>
    <vmport state="off"/>
  </features>
  <cpu mode="host-model" check="partial">
    <model fallback="allow"/>
    <topology sockets="1" cores="4" threads="2"/>
  </cpu>
  <clock offset="localtime">
    <timer name="rtc" tickpolicy="catchup"/>
    <timer name="pit" tickpolicy="delay"/>
    <timer name="hpet" present="no"/>
    <timer name="hypervclock" present="yes"/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled="no"/>
    <suspend-to-disk enabled="no"/>
  </pm>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type="file" device="disk">
      <driver name="qemu" type="qcow2"/>
      <source file="/home/mortivarus/Documents/qemu/win10"/>
      <target dev="vda" bus="virtio"/>
      <address type="pci" domain="0x0000" bus="0x03" slot="0x00" function="0x0"/>
    </disk>
    <disk type="file" device="cdrom">
      <driver name="qemu" type="raw"/>
      <source file="/home/mortivarus/Downloads/Win10_21H1_EnglishInternational_x64.iso"/>
      <target dev="sdb" bus="sata"/>
      <readonly/>
      <address type="drive" controller="0" bus="0" target="0" unit="1"/>
    </disk>
    <disk type="file" device="cdrom">
      <driver name="qemu" type="raw"/>
      <source file="/home/mortivarus/Documents/qemu/virtio-win-0.1.196.iso"/>
      <target dev="sdc" bus="sata"/>
      <readonly/>
      <address type="drive" controller="0" bus="0" target="0" unit="2"/>
    </disk>
    <controller type="usb" index="0" model="qemu-xhci" ports="15">
      <address type="pci" domain="0x0000" bus="0x02" slot="0x00" function="0x0"/>
    </controller>
    <controller type="pci" index="0" model="pcie-root"/>
    <controller type="pci" index="1" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="1" port="0x10"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x0" multifunction="on"/>
    </controller>
    <controller type="pci" index="2" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="2" port="0x11"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x1"/>
    </controller>
    <controller type="pci" index="3" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="3" port="0x12"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x2"/>
    </controller>
    <controller type="pci" index="4" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="4" port="0x13"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x3"/>
    </controller>
    <controller type="pci" index="5" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="5" port="0x14"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x4"/>
    </controller>
    <controller type="pci" index="6" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="6" port="0x8"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0" multifunction="on"/>
    </controller>
    <controller type="pci" index="7" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="7" port="0x9"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x1"/>
    </controller>
    <controller type="pci" index="8" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="8" port="0xa"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x2"/>
    </controller>
    <controller type="pci" index="9" model="pcie-to-pci-bridge">
      <model name="pcie-pci-bridge"/>
      <address type="pci" domain="0x0000" bus="0x05" slot="0x00" function="0x0"/>
    </controller>
    <controller type="sata" index="0">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1f" function="0x2"/>
    </controller>
    <interface type="network">
      <mac address="52:54:00:6b:3f:e5"/>
      <source network="default"/>
      <model type="e1000e"/>
      <address type="pci" domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
    </interface>
    <serial type="pty">
      <target type="isa-serial" port="0">
        <model name="isa-serial"/>
      </target>
    </serial>
    <console type="pty">
      <target type="serial" port="0"/>
    </console>
    <input type="tablet" bus="usb">
      <address type="usb" bus="0" port="1"/>
    </input>
    <input type="mouse" bus="ps2"/>
    <input type="keyboard" bus="ps2"/>
    <sound model="ich9">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1b" function="0x0"/>
    </sound>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
      </source>
      <rom file="/home/mortivarus/Documents/qemu/MSI.GTX1070.8192.161024_patched.rom"/>
      <address type="pci" domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x1"/>
      </source>
      <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x00" slot="0x14" function="0x0"/>
      </source>
      <address type="pci" domain="0x0000" bus="0x09" slot="0x01" function="0x0"/>
    </hostdev>
    <memballoon model="virtio">
      <address type="pci" domain="0x0000" bus="0x04" slot="0x00" function="0x0"/>
    </memballoon>
  </devices>
</domain>

````

