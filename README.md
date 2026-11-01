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

### The setup
```sh
# Is virtualization supported by CPU
egrep -c '(vmx|svm)' /proc/cpuinfo

# Install packages
sudo apt update
sudo apt install qemu-system-x86 libvirt-daemon-system \
libvirt-clients virt-manager virtinst cpu-checker

# Add the user to groups
sudo adduser "$USER" libvirt
sudo adduser "$USER" kvm

# Then delog / relog to the user session

# Check intallation
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
#### Permissions
```sh
# Check the rights of each node in the path
# Each dir needs 'x' bit for "others"
namei -l "$LOCATION/libvirt-images"
```

Get the latest stable Fedora.  
At the moment this is **Fedora Workstation 44**.  
```sh
cd /your/target/location
wget -P ~/Downloads https://download.fedoraproject.org/pub/fedora/linux/releases/44/Workstation/x86_64/iso/Fedora-Workstation-Live-44-1.7.x86_64.iso
virt-install --name fedora-ebpf --memory 6144 --vcpus 4 --disk size=40,format=qcow2 \
--cdrom ~/Downloads/Fedora-Workstation-Live-44-1.7.x86_64.iso \
--os-variant fedora-unknown --network network=default
```

