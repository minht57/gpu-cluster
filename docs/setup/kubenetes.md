# Kubernetes installation
This setup is applied for both a master node and worker node

## Installation
- Install SSH
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openssh-server ca-certificates curl gnupg lsb-release apt-transport-https -y

sudo service ssh start
```

- Disable swap and comment out the swap partition in `/etc/fstab`
```bash
sudo swapoff -a
```

- Docker installation
```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER
```

- Container runtime for Kubernetes
    - Reference [this repo](https://github.com/Mirantis/cri-dockerd).

            cd
            git clone https://github.com/Mirantis/cri-dockerd.git
            wget https://storage.googleapis.com/golang/getgo/installer_linux
            chmod +x ./installer_linux
            ./installer_linux
            source ~/.bash_profile
            cd cri-dockerd
            mkdir bin
            go build -o bin/cri-dockerd
            mkdir -p /usr/local/bin

            sudo install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
            sudo cp -a packaging/systemd/* /etc/systemd/system

            sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

            sudo systemctl daemon-reload
            sudo systemctl enable cri-docker.service
            sudo systemctl enable --now cri-docker.socket

- Deploy GPU plugin
    - Reference [this repo](https://github.com/NVIDIA/k8s-device-plugin#preparing-your-gpu-nodes)
    - Install the `nvidia-container-toolkit`

            distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
            curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey | sudo apt-key add -
            curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | sudo tee /etc/apt/sources.list.d/libnvidia-container.list
            sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit

    - Configure `docker`: `sudo vi /etc/docker/daemon.json`

            ```json
            {
                "default-runtime": "nvidia",
                "runtimes": {
                    "nvidia": {
                        "path": "/usr/bin/nvidia-container-runtime",
                        "runtimeArgs": []
                    }
                }
            }
            ```

    - Restart `docker`:
    
            sudo systemctl restart docker
    
    - Configure `containerd`: `sudo vi /etc/containerd/config.toml`
        - Comment out `disabled_plugins = ["cri"]`

                version = 2
                [plugins]
                  [plugins."io.containerd.grpc.v1.cri"]
                    [plugins."io.containerd.grpc.v1.cri".containerd]
                      default_runtime_name = "nvidia"

                      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
                        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
                          privileged_without_host_devices = false
                          runtime_engine = ""
                          runtime_root = ""
                          runtime_type = "io.containerd.runc.v2"
                          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
                            BinaryName = "/usr/bin/nvidia-container-runtime"

    - Restart `containerd`:

            sudo systemctl restart containerd

    - On MASTER only
  
            kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.0/nvidia-device-plugin.yml

- Kubernetes:
    - Reference [this link](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

            # ONE LINE
            curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

            # ONE LINE
            echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

            sudo apt-get update --allow-unauthenticated
            sudo apt-get install -y kubelet kubeadm kubectl --allow-unauthenticated
            sudo apt-mark hold kubelet kubeadm kubectl

- NFS
```bash
sudo apt install nfs-common
```

- Reboot

## kube master setup
These steps is set up in the master node ONLY.

- `kube` init
```bash
sudo kubeadm init --cri-socket=unix:///var/run/cri-dockerd.sock --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=172.16.0.16
```

- To check config:
```bash
sudo cat /etc/kubernetes/kubelet.conf
```
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
- Flannel network plugin:
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

- Allow pods to schedule on master:
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Helm installation
- Reference [this link](https://helm.sh/docs/intro/install/).
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## Setup new nodes
- Prerequisite:
    - Need internet for setup. Just be on Kube network should be ok
    - Install Ubuntu 22.04 + NVIDIA GPU driver (included with Ubuntu)
    - Install Kubernetes

- Run command on **Master**
```bash
kubeadm token create --print-join-command
```

- Reset Kube on new nodes:
```bash
sudo kubeadm reset --cri-socket=unix:///var/run/cri-dockerd.sock
```
- Clear old configurations
```bash
sudo -i
kubeadm reset
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
ip link set cni0 down
ip link delete cni0 type bridge
systemctl stop kubelet
systemctl stop docker
iptables --flush
iptables -tnat --flush
systemctl start kubelet
systemctl start docker
exit
```

- Run the generated command on new nodes:
    - NOTE: Note the need to add `--cri-socket=unix:///var/run/cri-dockerd.sock` to the command
```bash
sudo XXXX --cri-socket=unix:///var/run/cri-dockerd.sock
```

- Checking
    - Run `kubectl get nodes -o wide` to get the status of all nodes
    - GPU functionality check & count:

            kubectl get nodes -o=custom-columns=NAME:.metadata.name,GPUs:.status.capacity.'nvidia\.com/gpu'
