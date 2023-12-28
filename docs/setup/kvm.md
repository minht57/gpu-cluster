# KVM (Kernel-based Virtual Machine)
## Information
- Ubuntu server 22.04
- Name convension: 
    - VM node: cluster-vgn[1/2/3/..]
        - vgn: virtual GPU node
- IP addresses:
    - Virtual Node:
        - br0: 172.20.0.[100 + i]
        - br1: 192.168.33.[100 + i]
        - br2: 172.16.0.[100 + i]

## Host machine
### Install kvm
- Update
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install bridge-utils qemu-kvm virtinst libvirt-daemon virt-manager -y
```

- Check KVM
```bash
admin@cluster-gn1:~$ kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
```

- Check `libvirtd.service`
```bash
admin@cluster-gn1:~$ sudo systemctl status libvirtd.service 
[sudo] password for admin: 
● libvirtd.service - Virtualization daemon
     Loaded: loaded (/lib/systemd/system/libvirtd.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2023-06-13 03:40:55 UTC; 1 day 23h ago
TriggeredBy: ● libvirtd-admin.socket
             ● libvirtd-ro.socket
             ● libvirtd.socket
       Docs: man:libvirtd(8)
             https://libvirt.org
   Main PID: 2806 (libvirtd)
      Tasks: 23 (limit: 32768)
     Memory: 56.8M
        CPU: 13.964s
     CGroup: /system.slice/libvirtd.service
             ├─2806 /usr/sbin/libvirtd
             ├─3157 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/br0.conf --leasefile-ro --dhc>
             └─3158 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/br0.conf --leasefile-ro --dhc>

Jun 13 03:40:56 cluster-gn1 dnsmasq[3157]: reading /etc/resolv.conf
Jun 13 03:40:56 cluster-gn1 dnsmasq[3157]: using nameserver 127.0.0.53#53
Jun 13 03:40:56 cluster-gn1 dnsmasq[3157]: read /etc/hosts - 7 addresses
Jun 13 03:40:56 cluster-gn1 dnsmasq[3157]: read /var/lib/libvirt/dnsmasq/br0.addnhosts - 0 addresses
Jun 13 03:40:56 cluster-gn1 dnsmasq-dhcp[3157]: read /var/lib/libvirt/dnsmasq/br0.hostsfile
Jun 13 03:40:56 cluster-gn1 libvirtd[2806]: libvirt version: 8.0.0, package: 1ubuntu7.5 (Marc Deslauriers <>
Jun 13 03:40:56 cluster-gn1 libvirtd[2806]: hostname: cluster-gn1
Jun 13 03:40:56 cluster-gn1 libvirtd[2806]: Tried to update an unsupported keyword YA: skipping.
Jun 13 03:40:56 cluster-gn1 libvirtd[2806]: Tried to update an unsupported keyword YA: skipping.
```

### Config network
- Create a `br0.xml` file
```xml
<network>
  <name>br0</name>
  <uuid>427c954b-4993-4554-bb1d-79d99882a9a6</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='br0' stp='on' delay='0'/>
  <mac address='52:54:00:0d:ff:76'/>
  <ip address='172.20.0.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='172.20.0.2' end='172.20.0.254'/>
    </dhcp>
  </ip>
</network>
```

- Create network
```bash
virsh net-define br0.xml
virsh net-start br0
virsh net-autostart br0
```

- Check network
```bash
aimc-hpc-gn4@aimc-hpc-gn4:~/kvm$ virsh net-list
 Name   State    Autostart   Persistent
-----------------------------------------
 br0    active   yes         yes
```

- These next commands will delete the default private network, this is not required but you can if you prefer to delete it.
```bash
virsh net-destroy default
virsh net-undefine default
```

- Restart  libvirt daemon
```bash
sudo systemctl restart libvirtd.service
```

### Create VMs
- Create an `images` folder
```bash
sudo mkdir -p /mnt/local/kvm/images
```

- Script details
```bash
sudo virt-install --name vgn1 \
--os-variant ubuntu22.04 \
--vcpus 18 \
--memory 215040 \
--location /mnt/local/kvm/ubuntu-22.04.2-live-server-amd64.iso,kernel=casper/vmlinuz,initrd=casper/initrd \
--network bridge=br0,model=virtio \
--network bridge=br1,model=virtio \
--network bridge=br2,model=virtio \
--disk /mnt/local/kvm/images/vgn1.qcow2,size=150 \
--graphics vnc \
--extra-args='console=ttyS0,115200n8 --- console=ttyS0,115200n8' \
--debug
```

- Please change the appropriate name in the script (arguments: `--name` and `--disk`)
- This will waiting for finishing installtion from GUI
### Open VNC to setup
- Create an SSH tunnel
```bash
ssh admin-gn1@192.168.33.11 -NfL 5900:127.0.0.1:5900
```

- Download VNC from https://www.realvnc.com/en/connect/download/viewer/
- Install
- Open VM using this URL: 127.0.0.1:5900
- Install as normal installation


### Finish the creation of a new VM
- After choosing `Reboot`, open another terminal and SSH to the native machine (`aimc-en1` in this case)
  - Go in the console: `virsh console ven1`
  - Press `Enter` to reboot the VM
  - Press `Ctrl` + `]` to exit

## New VMs
### Set up network
- Create and edit the config file
```bash
vi /etc/netplan/00-installer-config.yaml
```

- Choose the IP address from `x.x.x.[100 + VM ID]`
```yaml
network:
  ethernets:
    enp1s0:
      addresses:
      - 172.20.0.101/24
      nameservers:
        addresses: []
        search: []
      routes:
      - to: default
        via: 172.20.0.1
    enp2s0:
      addresses:
      - 172.16.0.101/24
      nameservers:
        addresses: []
        search: []
      routes:
      - to: default
        via: 172.16.0.1
    enp20s0:
        addresses:
        - 192.168.33.101/24
        nameservers:
          addresses: []
        routes:
          - to: default
            via: 192.168.33.1
  version: 2
```

- Apply change
```bash
sudo netplan generate
sudo netplan apply
```

## References
- [https://www.wpdiaries.com/kvm-on-ubuntu/](https://www.wpdiaries.com/kvm-on-ubuntu/)
- [https://www.wpdiaries.com/ubuntu-on-kvm/](https://www.wpdiaries.com/ubuntu-on-kvm/)
- [https://deploy.equinix.com/developers/guides/kvm-and-libvirt](https://deploy.equinix.com/developers/guides/kvm-and-libvirt)
