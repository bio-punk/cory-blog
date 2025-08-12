---
title: "带IB和GPU的docker安装"
date: 2025-08-11T15:03:39+08:00
draft: false
description: ""
tags: ["container", "build"]
---

# Docker带IB和GPU安装
## 参考

## 流程
1. 安装docker
```bash
#!/bin/bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
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
#!/bin/bash
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

3. 主机sshd配置  
修改 ```/etc/ssh/sshd_config```以允许远程主机通过ssh隧道向容器内提供服务

```
GatewayPorts clientspecified
```

4. docker网络配置   
配置 ```/etc/systemd/system/docker.service.d/http-proxy.conf``` 以允许docker通过代理服务拉取镜像
```conf
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7897"
Environment="HTTPS_PROXY=http://127.0.0.1:7897"
Environment="NO_PROXY=localhost,127.0.0.1"

```

配置 ```~/.docker/config.json``` 以允许容器内使用宿主机的代理服务,其中```MASTER_ADDR```应设置为 ```ip a|grep bond0|grep 24|awk '{print $2}'|awk -F/ '{print $1}'```获取的值

```json
{
 "proxies": {
   "default": {
     "httpProxy": "http://${MASTER_ADDR}:7897",
     "httpsProxy": "http://${MASTER_ADDR}$:7897",
     "noProxy": "localhost,127.0.0.1/8,10.20.200.1/8"
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

5. MTU配置  
还是修改 ```~/.docker/config.json```
配置合理的MTU以便Docker通过宿主机联接互联网

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

6. overlay网络创建
## 头结点配置
```bash
export MASTER_ADDR="10.20.20.11"
docker swarm init --advertise-addr $MASTER_ADDR
# docker swarm join --token SWMTKN-1-4f4899ad7l558ida8qet6jgswuyqa3nli89udora7warw302qh-6gu0scglazcoy11p9bqumxnor 10.20.20.11:2377
docker network create -d overlay --subnet="10.20.200.0/24" --gateway="10.20.200.1" --attachable overlay01
# nktlph09p0f8e9fcdqotw15d8
```
## 从节点配置
```bash
docker swarm join --token SWMTKN-1-4f4899ad7l558ida8qet6jgswuyqa3nli89udora7warw302qh-6gu0scglazcoy11p9bqumxnor 10.20.20.11:2377
```

6. 配置NGC密钥
登录(NGC)[https://org.ngc.nvidia.com/]  
在setup>API Keys中点击 Generate Personal Key获取密钥  
```bash
docker login nvcr.io
Username:$oauthtoken
Password:XXX
```
6. 运行NCCL-tests
## Dockerfile
```dockerfile
ARG NGC_VERSION=23.04-py3
FROM nvcr.io/nvidia/pytorch:${NGC_VERSION} AS base
ENV DEBIAN_FRONTEND=noninteractive
RUN mkdir /run/sshd && apt-get update && apt-get install -y \
    openssh-server \
    openssh-client \
    vim \
    wget \
    git
RUN git clone https://github.com/NVIDIA/nccl-tests.git /nccl-tests && cd /nccl-tests && make MPI=1 MPI_HOME=/opt/hpcx/ompi/ -j
ENV TZ=Asia/Shanghai
ENV NCCL_IB_RETRY_CNT="13" NCCL_IB_TIMEOUT="22" \
    NCCL_DEBUG="WARN" \
    NCCL_IB_HCA="mlx5_0:1,mlx5_1:1,mlx5_4:1,mlx5_5:1" \
    NCCL_IB_P2P_DISABLE="0" \
    NCCL_IB_DISABLE="0"
RUN apt install -y net-tools sudo
RUN dpkg-statoverride --remove /usr/lib/dbus-1.0/dbus-daemon-launch-helper || true
```
## 制作本地docker文件
```bash
docker pull nvcr.io/nvidia/pytorch:23.04-py3
docker pull nvcr.io/nvidia/pytorch:25.06-py3
docker save -o pytorch.tar nvcr.io/nvidia/pytorch:23.04-py3
docker build -t nccl-test:v2025.08.12 .
docker save -o /data/dockerimage/nccl-test_2025.08.12.tar nccl-test:v2025.08.12
```
## 1号节点执行
```bash
docker run -itd --runtime=nvidia --gpus all --device=/dev/infiniband --shm-size 1024G --ulimit memlock=-1  --network overlay01 \
  --ip 10.20.200.100 \
  -v /data:/data -v /root:/root -v /root/.ssh/:/root/.ssh/ c81e26d88f9b
EX_ID=`docker ps --latest -q`
```

## 其余节点执行
```bash
docker load -i /data/dockerimage/pytorch_23.04-py3.tar
docker load -i /data/dockerimage/nccl-test_2025.08.12.tar
docker run -itd --runtime=nvidia --gpus all --device=/dev/infiniband --shm-size 1024G --ulimit memlock=-1  --network overlay01 \
  --ip 10.20.200.$nodeid \
  -v /data:/data -v /root:/root -v /root/.ssh/:/root/.ssh/ c81e26d88f9b
EX_ID=`docker ps --latest -q`
docker exec -it ${EX_ID} bash /usr/sbin/sshd
```

## 节点互通测试
```bash
ssh -T 10.20.200.101
```

## 启动nccl-tests
```bash
EX_ID=`docker ps --latest -q`
docker exec -it ${EX_ID} bash
cd cd /data/apps/nccl-tests-2.16.7_ubuntu2002/
mpirun --allow-run-as-root \
  -np 32 -H 10.20.200.100:8,10.20.200.101:8,10.20.200.102:8,10.20.200.103:8 \
  ./build/all_reduce_perf -g 1 -b 512M -e 16G -f 2 2>&1 |tee test4node.log
```

{{<gist bio-punk ddd0800683b8b21589f88c84906451ef>}}