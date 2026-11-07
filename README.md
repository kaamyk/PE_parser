# PE_parser

## The environment
### The tools and vocabulary :
**QEMU** (https://www.qemu.org/) as emulator :  
It is a generic and open source machine emulator and virtualizer.
It works with Kernel-based Virtual Machine (KVM) it can run instruction directly in the virtual machine's CPU.  
  
**Fedora** (https://fedoraproject.org) as VM's OS :  
Fedora is a Linux distribution focused on the the latest open-source technologies while being stable.  
It has a BPF Type Format (BTF) by default.  

**BTF** : 
BTF describes the the data types, functions and eBPF's and kernel's structures.

**ACL** :
*Definition to do*
### The setup
```sh
# Is virtualization supported by CPU
egrep -c '(vmx|svm)' /proc/cpuinfo

# Install packages
sudo apt update
sudo apt install qemu-system-x86 libvirt-daemon-system \
libvirt-clients virt-manager virtinst cpu-checker virt-viewer

# Add the user to groups
sudo adduser "$USER" libvirt
sudo adduser "$USER" kvm

# Then delog / relog to the user session

# Check installation
kvm-ok                  # expected : KVM acceleration can be used
virsh list --all        # expected : empty, no error
sudo virsh net-list --all    # expected : default network active. sudo needed for network

# If network inactive
virsh net-start default
virsh net-autostart default
```

### Create the VM
#### Create the storage pool (optional)
```sh
# Make sure virsh command talks to the right libvrt instance
export LIBVIRT_DEFAULT_URI=qemu:///system
LOCATION=/the/location
virsh pool-define-as ssdpool dir --target "$LOCATION/libvirt-images"
# Create the directory
virsh pool-build poolName
virsh pool-start poolName
virsh pool-autostart poolName
# Check the available space
# The size shown is not allocated, it is just usable for futur machines
virsh pool-info poolName
```
#### Network
Check if network started:
```sh
virsh -c qemu:///system net-list --all
```
Start the network:
```sh
virsh -c qemu:///system net-start default
```
Set autostart:
```sh
virsh -c qemu:///system net-autostart default
```
Now check again.  
#### Permissions
Check if the kernel handles ACL:
```sh
grep ACL /boot/config-$(uname -r)
```
If ACL specified the 'ls -l' output has a '+':
```sh
ls -l file
```
Check the rights of each node in the path:
```sh
# Each dir needs 'x' bit for "others"
namei -l "$LOCATION/libvirt-images"
# getacl prints the permisions:
getacl file
```

Give the ACL permissions:
```sh
sudo setfacl -m u:libvirt-qemu:x /location/of/pool
```

#### Console view
sudo apt install virt-viewer

### Installation
Get the latest stable Fedora.  
At the moment this is **Fedora Workstation 44**.  
```sh
wget -P ~/Downloads https://download.fedoraproject.org/pub/fedora/linux/releases/44/Workstation/x86_64/iso/Fedora-Workstation-Live-44-1.7.x86_64.iso
cd /your/target/location
sudo virt-install --connect qemu:///system --name fedora-ebpf \
  --memory 6144 --vcpus 4 --disk pool=ssdpool,size=40,format=qcow2 \
  --cdrom /mnt/hdd-storage/ISOs/Fedora-Workstation-Live-44-1.7.x86_64.iso \
  --os-variant fedora-unknown --network network=default
```
In the launched VM, follow the instruction to fully install the OS on the virtual disk.  
Stop the VM:
```sh
virsh -c qemu:///system start fedora-ebf
```
Check that the ISO is no longer attached:
```sh
# from host
virsh -c qemu :///system domblklist fedora-ebpf
# if not
virsh -c qemu:///system change-media fedora-ebpf sda --eject --config

# Start the VM
virsh -c qemu:///system start fedora-ebf
# in VM
# Update & update SSH
sudo dnf upgrade --refresh -y
sudo reboot
```
After the reboot:
```sh 
uname -r													# Note the kernel version
sudo systemctl enable --now sshd
systemctl is-active sshd					# expected: active
ip -4 addr show										# Note the IP address
```
From the host:
```sh
virsh -c qemu:///system domifaddr fedora-ebpf
ssh bibi@<IP_ADDR>
```
The VM is setup, let's save it.
```sh
virsh -c qemu:///system snapshot-create-as fedora-ebpf --name base-updated
virsh -c qemu:///system snapshot-list fedora-ebpf
virsh -c qemu:///system start fedora-ebpf
```

### Install packages
```sh
# Required
sudo dnf install -y clang llvm libbpf-devel bpftool kernel-devel-$(uname -r) \
elfutils-libelf-devel zlib-devel \
gcc make git pkgconf-pkg-config
# Optional 
sudo dnf install -y strace jq tmux nmap-ncat perf


# Verify the installation
# must match the output of uname -r
rpm -q kernel-devel
# must exist: it is the BTF info CO-RE relies on
ls -l /sys/kernel/btf/vmlinux
clang --version
# expected: bpf, bpfel, bpfeb
clang --print-targets | grep -i bpf
# should mention libbpf and the features
bpftool version
# libbpf version
pkg-config --modversion libbpf
```

### Generate vmlinux header file
```sh
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```
