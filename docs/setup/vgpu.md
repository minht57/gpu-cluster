# vGPU
## Install dependencies
### Install dependencies
```bash
sudo apt install unzip
```

### Download driver
- Download package `NVIDIA-GRID-Ubuntu-KVM-525.85.07-525.85.05-528.24.zip`
- Create token for licensing
- Extract
```bash
mkdir -p /mnt/nfs/vGPU
unzip NVIDIA-GRID-Ubuntu-KVM-525.85.07-525.85.05-528.24.zip -d /mnt/nfs/vGPU
```

### Remove current NVIDIA driver if existing
```bash
bash /mnt/local/NVIDIA-Linux-x86_64-450.216.04.run --uninstall
```

## Host machine (compute node)
### Disable Nouveau and enable nvfio
```bash
echo -e "vfio\nvfio_iommu_type1\nvfio_pci\nvfio_virqfd" >> /etc/modules
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
update-initramfs -u -k all
```

### Disable X server
```bash
sudo init 3
```
- Ref: [link](https://unix.stackexchange.com/questions/25668/how-to-close-x-server-to-avoid-errors-while-updating-nvidia-driver)

### Install driver
```bash
cd /mnt/nfs/vGPU/Host_Drivers
sudo apt install ./nvidia-vgpu-ubuntu-525_525.85.07_amd64.deb
```
- Ref: [link](https://docs.nvidia.com/grid/15.0/grid-vgpu-user-guide/index.html#ubuntu-install-configure-vgpu)

### Configuring vGPU [Deprecated]
- Check domain/bus/slot/function
```bash
admin@cluter-gn1:~$ lsmod | grep vfio
nvidia_vgpu_vfio       65536  272
mdev                   28672  1 nvidia_vgpu_vfio

admin@cluter-gn1:~$ lspci | grep NVIDIA
89:00.0 3D controller: NVIDIA Corporation GV100GL [Tesla V100 SXM2 32GB] (rev a1)
8a:00.0 3D controller: NVIDIA Corporation GV100GL [Tesla V100 SXM2 32GB] (rev a1)
b2:00.0 3D controller: NVIDIA Corporation GV100GL [Tesla V100 SXM2 32GB] (rev a1)
b3:00.0 3D controller: NVIDIA Corporation GV100GL [Tesla V100 SXM2 32GB] (rev a1)

admin@cluter-gn1:~$ virsh nodedev-list --cap pci| grep 89_00_0
pci_0000_89_00_0

admin@cluter-gn1:~$ virsh nodedev-dumpxml pci_0000_89_00_0 | egrep 'domain|bus|slot|function'
    <domain>0</domain>
    <bus>137</bus>
    <slot>0</slot>
    <function>0</function>
```

### Create vGPU manually
- Change to the root user
```bash
sudo -i
```

- Seach nvidia profiles
    - `8Q` - 8G/vGPU
    - `16Q` - 16G/vGPU
```bash
cd /sys/bus/pci/devices/0000:89:00.0/mdev_supported_types
grep -l "V100DX-8Q" nvidia-*/name
```
- The output:
```bash
nvidia-197/name
```

- Check available instances, the result should be greater than 0
```bash
cat nvidia-197/available_instances
```

- Generate uuid for the vGPU
```bash
# uuidgen
b87b1cd3-feb8-4ca6-88af-33b3c9f81425
```

- Write the UUID that you obtained in the previous step to the `create` file in the registration information directory for the vGPU type that you want to create 
```bash
echo "b87b1cd3-feb8-4ca6-88af-33b3c9f81425" > nvidia-197/create
```

- Make the mdev device file that you created to represent the vGPU persistent.
```bash
mdevctl define --auto --uuid b87b1cd3-feb8-4ca6-88af-33b3c9f81425
```

- Confirm that the vGPU was created
```bash
ls -l /sys/bus/mdev/devices/
```
or
```bash
mdevctl list
```

### Create all vGPU
```bash
cd /mnt/nfs/vGPU
sudo ./admin-create-all-vgpus.sh -r 8
```
- `-r 8`: choose 8GB of vGPU memory

### Adding One or More vGPUs to a Linux with KVM Hypervisor VM by Using virsh
```bash
virsh edit vgn1
```

- Add device entries
```xml
<device>
...
    <hostdev mode='subsystem' type='mdev' model='vfio-pci'>
      <source>
        <address uuid=''/>
      </source>
    </hostdev>
    <hostdev mode='subsystem' type='mdev' model='vfio-pci'>
      <source>
        <address uuid=''/>
      </source>
    </hostdev>
</device>
```

- Start/Restart the VM
```bash
virsh start vgn1
```

## VMs
### Access to the VM via `virsh`
```bash
admin@cluter-gn1:/mnt/nfs/vGPU$ virsh list --all
 Id   Name   State
----------------------
 2    vgn1   running

admin@cluter-gn1:/mnt/nfs/vGPU$ virsh console vgn1
Connected to domain 'vgn1'
Escape character is ^] (Ctrl + ])

admin@cluter-vgn1:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:05:41:79 brd ff:ff:ff:ff:ff:ff
    inet 172.20.0.101/24 brd 172.20.0.255 scope global enp1s0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe05:4179/64 scope link 
       valid_lft forever preferred_lft forever
3: enp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:89:d9:16 brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.41/24 brd 172.16.0.255 scope global enp2s0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe89:d916/64 scope link 
       valid_lft forever preferred_lft forever
4: enp20s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:51:d3:51 brd ff:ff:ff:ff:ff:ff
    inet 192.168.33.101/24 brd 192.168.33.255 scope global enp20s0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe51:d351/64 scope link 
       valid_lft forever preferred_lft forever
```

- Install SSH server
```bash
sudo apt install openssh-server
```

- Exit console: `Ctrl + ]`

### Access to the VM via `ssh`
- Copy ssh key
```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@172.16.0.101
```

- SSH to the VM
```bash
ssh ubuntu@172.16.0.101
```

- Copy NVIDIA Guest drive and token
```bash
scp /mnt/nfs/vGPU/Guest_Drivers/nvidia-linux-grid-525_525.85.05_amd64.deb ubuntu@172.16.0.101:/home/admin
scp /mnt/nfs/vGPU/client_configuration_token_03-23-2023-09-07-06.tok ubuntu@172.16.0.101:/home/admin
```

- Install NVIDIA driver
```bash
sudo apt install ./nvidia-linux-grid-525_525.85.05_amd64.deb
```

- Change FeatureType from 0 to 2
```bash
sudo nano /etc/nvidia/gridd.conf
```
```
# Description: Set Feature to be enabled
# Data type: integer
# Possible values:
#    0 => for unlicensed state
#    1 => for NVIDIA vGPU (Optional, autodetected as per vGPU type)
#    2 => for NVIDIA RTX Virtual Workstation
#    4 => for NVIDIA Virtual Compute Server
# All other values reserved
FeatureType=2
```

- Restart VM
```bash
sudo reboot
```

- SSH to VM
- Copy token
```bash
sudo cp client_configuration_token_03-23-2023-09-07-06.tok /etc/nvidia/ClientConfigToken/
```

- Change mode
```bash
chmod 744 /etc/nvidia/ClientConfigToken/client_configuration_token_03-23-2023-09-07-06.tok
```

- Restart nvidia-gridd deamon
```bash
sudo systemctl restart nvidia-gridd.service
```

- Test
```bash
nvidia-smi -q
```

- Rebooting the VM may save your time when the vGPU does not recognize the license
```bash
sudo reboot
```

