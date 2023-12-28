# Native machines
## Information
- Ubuntu server/desktop 22.04 (server version is preferred)
- Name convension: 
    - Headnode: cluster-hn
    - Compute node: cluster-gn[1/2/3/..]
        - gn: GPU node
- NFS folder
    - NFS server is hosted on the Headnode
- IP addresses:
    - Headnode:
        - br1: 192.168.33.5
        - br2: 172.16.0.5
    - Compute node:
        - br0: 172.20.0.[10 + Node ID]
        - br1: 192.168.33.[10 + Node ID]
        - br2: 172.16.0.[10 + Node ID]


## Download and install Ubuntu
- Download and install Ubuntu on the native machine

## Set up networking
There are 3 networks:
- br0: internal communication of virtual machines (setup later)
- br1: high-speed network (bonding config)
- br2: internet access (1/10GB network)

### Set up bonding
- Check hardware
```bash
sudo lshw -C network
```
- Install
```bash
ifconfig ens11 down
ifconfig ens12 down

ip link add bond0 type bond mode 802.3ad

ip link set ens11 master bond0
ip link set ens12 master bond0
```

### Create a bridge br1 (100G network) and a bridge br2 (1/10G network)
- Create a new file located at `/etc/netplan/01-netcfg.yaml`
```yaml
network:
  ethernets:
    ens11:
      dhcp4: false
      dhcp6: false
    ens12:
      dhcp4: false
      dhcp6: false
  bonds:
    bond0:
      dhcp4: false
      interfaces:
        - ens11
        - ens12
      parameters:
        mode: balance-tlb
        mii-monitor-interval: 100
  version: 2
  bridges:
    br1:
      dhcp4: false
      dhcp6: false
      interfaces: [ bond0 ]
      addresses: [172.16.0.5/24]
      nameservers:
         addresses: []
      routes:
         - to: default
           via: 172.16.0.1
      mtu: 1500
      parameters:
        stp: true
        forward-delay: 4
    br2:
      dhcp4: false
      dhcp6: false
      interfaces: [ eno1 ]
      addresses: [192.168.33.5/24]
      nameservers:
         addresses: []
      routes:
         - to: default
           via: 192.168.33.1
      mtu: 1500
      parameters:
        stp: true
        forward-delay: 4
```
- Appyly change
```bash
sudo netplan generate
sudo netplan apply
```

### Check bonding
```bash
# after setting bonding, [bonding] is loaded automatically
lsmod | grep bond

ip address show
ethtool bond0
```

## Generate an SSH key
```bash
ssh-keygen -t rsa
```

## Set up the NFS


### Server
- Create a new mount point
```bash
sudo mkdir /mnt/nfs-hn
```
- Add line `/dev/sdb /mnt/nfs-hn ext4 defaults 0 0` in `/etc/fstab`

- Install NFS server
```bash
sudo apt install nfs-kernel-server
sudo chown nobody:nogroup /mnt/nfs-ehn1
```

- Edit exports `sudo nano /etc/exports`
```bash
/mnt/nfs-hn/      172.16.0.0/24(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
```

- Restart NFS server
```bash
sudo systemctl restart nfs-kernel-server
```

### Client
This is set up in all machines with a required access to the NFS folder.

- Install NFS client
```bash
sudo apt install nfs-common
```

- Create folders
```bash
sudo mkdir -p /mnt/nfs
sudo mkdir -p /mnt/local
```
- Edit `fstab` file: `sudo vi /etc/fstab`
```bash
/dev/sdb /mnt/local ext4 defaults 0 0
172.16.0.5:/mnt/nfs-hn/   /mnt/nfs  nfs  defaults 1 2
```
- Remount
```bash
sudo mount -av
```

