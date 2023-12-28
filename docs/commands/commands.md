# Usefull commands
## KVM
- List all VMs
```bash
virsh list --all
```

- Start VM
```bash
virsh start cluster-vgu1
```

- Shutdown VM
```bash
virsh shutdown cluster-vgu1
```

- Force to shutdown
```bash
virsh destroy cluster-vgu1
```

- List all IP address
```bash
virsh net-dhcp-leases br0
```

- Access serial terminal
  - To exit: Ctrl + ]
```bash
virsh console cluster-vgu1
```

- Remove VM and delete its storage
```bash
virsh undefine cluster-vgu1 --remove-all-storage
```

- Change VM name
```bash
virsh domrename cluster-vgu1 cluster-vgu2
```

- Check state
```bash
virsh domstate cluster-vgu1
```

- Export config
```bash
virsh dumpxml cluster-vgu5 > aimc-template.xml
```

- Check virtual disk
```bash
virsh domblklist cluster-vgu5
```

- Define new VM
```bash
virsh define aimc-template.xml
```
NOTE: If it reports the error that the CPU is not compatible with host CPU, then the CPU of the VM needs to be modified. Replace CPU configuration with `<cpu mode='host-passthrough' check='none'/>`
  ```bash
  virsh edit guest_name
  ```

- Clone VM:
```bash
virt-clone --original cluster-vgu1 --name cluster-vgu1  --file /mnt/local/kvm/images/cluster-vgu2.qcow2
```

- vGPU template
```xml
    <hostdev mode='subsystem' type='mdev' managed='no' model='vfio-pci' display='off'>
      <source>
        <address uuid='26216876-d01a-4eed-b06d-341abcf4da00'/>
      </source>
    </hostdev>
```

## Change hostname
```bash
sudo nano /etc/hostname

sudo nano /etc/hosts

sudo hostname cluster-vgu5

sudo nano /etc/netplan/00-installer-config.yaml

reboot
```

## To prevent a node from scheduling new pods use:
```bash
kubectl cordon <node-name>

kubectl uncordon <node-name>
```

## Resize VM disk
### On host
```bash
ls -al /mnt/nfs-en2/kvm/images/cluster-vgu1.qcow2 
sudo qemu-img info  /mnt/nfs-en2/kvm/images/cluster-vgu1.qcow2
sudo qemu-img resize  /mnt/nfs-en2/kvm/images/cluster-vgu1.qcow2 +50G
sudo qemu-img info  /mnt/nfs-en2/kvm/images/cluster-vgu1.qcow2
sudo fdisk -l /mnt/nfs-en2/kvm/images/cluster-vgu1.qcow2
```

### On VM
```bash
lsblk
sudo pvs
sudo apt install cloud-guest-utils
sudo growpart /dev/vda 3
sudo lsblk 
sudo pvresize /dev/vda3
sudo vgs
sudo lvextend -r -l +100%FREE  /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs  /dev/mapper/ubuntu--vg-ubuntu--lv
df -hT | grep mapper
```

## Generate ssh key
```bash
ssh-keygen -t rsa
```

## NFS
- if you cannot mount the nfs, try to add `ip route`
```bash
ip route add 172.0.0.0/8 via 172.20.0.1 dev enp1s0
showmount -e  172.16.0.5
mount 172.16.0.5:/mnt/nfs-en2/ /mnt/nfs
ls /mnt/nfs/
```

## Kube debug
```bash
# Nodes
kubectl get nodes --all-namespaces -o wide

# Pods
kubectl get pods -n hub -o wide

# Services
kubectl get services --all-namespaces

# Describe
kubectl describe pod <pod_name> -n hub

# Log
kubectl logs <pod_name> -n hub

# Create a new token
kubeadm token create --print-join-command

# Restart a namespaces
kubectl -n kube-system rollout restart deploy
```

## Check port
```bash
sudo lsof -i tcp
sudo netstat -ntlp
nc -z -v 172.16.0.5 8081
```

## Fail to load `nvidia-gridd.service` due to time issues
- If the pod is failed to pull new images, there may be have an issues of full disk --> Resize VM disk
- If `nvidia-gridd.service` have an issue related to clock, please set a realtime clock
```bash
timedatectl set-ntp off
timedatectl set-time '2023-06-20 16:14:50'
```
- If all vGPU are attached and `nvidia-smi -q` shows "Unlicensed", please check the permission of the `.tok` file (correct permission is `744`).

## Helm
```bash
helm show values jupyterhub/jupyterhub > values.yaml
```
