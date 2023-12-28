# Others
This includes some setups which may need in the system.

## Set up the network card driver (`mlnx_ofed`)
- Download [`mlnx_ofed`](https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/)

```bash
mkdir -p /home/cluster-gn1/network & cd /home/cluster-gn1/network
wget https://content.mellanox.com/ofed/MLNX_OFED-5.8-2.0.3.0/MLNX_OFED_LINUX-5.8-2.0.3.0-ubuntu22.04-x86_64.iso
```

- Install the driver
```bash
sudo -i

mount -o loop /home/cluster-gn1/network/MLNX_OFED_LINUX-5.8-2.0.3.0-ubuntu22.04-x86_64.iso /mnt
cd /mnt
./mlnxofedinstall --force
/etc/init.d/openibd restart

umount /mnt
```
