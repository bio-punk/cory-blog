---
title: "Docker_gpu_ib"
date: 2025-08-11T15:03:39+08:00
draft: true
description: ""
---

# Docker带IB和GPU安装
## 参考

## 流程
1. 安装docker
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
docker version
``` 

2. 配置NVIDIA_CONTAINER_TOOLKIT
```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID) && \
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg && \
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.17.8-1
sudo apt-get install -y \
    nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

3. docker网络配置
```conf
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7897"
Environment="HTTPS_PROXY=http://127.0.0.1:7897"
Environment="NO_PROXY=localhost,127.0.0.1"

```

```json
{
 "proxies": {
   "default": {
     "httpProxy": "http://127.0.0.1:7897",
     "httpsProxy": "http://127.0.0.1:7897",
     "noProxy": "localhost,127.0.0.1/8"
   }
 }
}
```

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
cp http-proxy.conf /etc/systemd/system/docker.service.d
mkdir -p /root/.docker
cp config.json /root/.docker/
sudo systemctl daemon-reload
sudo systemctl restart docker
docker info|grep -i proxy
```

4. MTU配置
还是修改 /root/.docker/config.json

```json
{   "registry-mirrors": ["https://docker.m.daocloud.io"],
    "mtu":1442,
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```
5. overlay网络创建
## 头结点配置
```bash
export MASTER_ADDR="10.20.20.11"
docker swarm init --advertise-addr $MASTER_ADDR
docker network create -d overlay --subnet="10.20.200.0/24" --gateway="10.20.200.1" --attachable overlay01

```