---
title: "B200 Vllm"
date: 2025-08-21T21:07:39+08:00
draft: true
description: ""
tags: ["build", "llm", "vllm"]
---

# 基于CUDA 12.9和SM_100的VLLM环境搭建

## 环境构建
### Container
基础镜像：以```nvcr.io/nvidia/pytorch:25.06-py3@06aa7e7a6f5a```为基础构建容器  
根据[pytorch-25.06 release info](https://docs.nvidia.com/deeplearning/frameworks/pytorch-release-notes/rel-25-06.html)
该镜像提供下列工具
|Name|version|
|---|---|
|CUDA|12.9.1|
|Torch-TensorRT|2.8.0a0|
|NVIDIA DALI®|1.50|
|nvImageCodec|0.2.0.7|
|MAGMA|2.6.2|
|JupyterLab|4.3.6|
|TensorRT Model Optimizer|0.29|
|TransformerEngine|2.4|
|NVIDIA RAPIDS™|25.04|
|NVIDIA cuBLASMp|0.4.0|

建议在此基础上构建镜像以便于提供服务  
例如：
```Dockerfile
ARG NGC_VERSION=25.06-py3
FROM nvcr.io/nvidia/pytorch:${NGC_VERSION} AS base
ENV DEBIAN_FRONTEND=noninteractive
RUN mkdir /run/sshd && apt-get update && apt-get install -y \
    openssh-server \
    openssh-client \
    vim \
    wget \
    git
ENV TZ=Asia/Shanghai
ENV NCCL_IB_RETRY_CNT="13" NCCL_IB_TIMEOUT="22" \
    NCCL_DEBUG="WARN" \
    NCCL_IB_DISABLE="0"
RUN apt install -y net-tools sudo
RUN dpkg-statoverride --remove /usr/lib/dbus-1.0/dbus-daemon-launch-helper || true
```
### 继续工作
#### sshd
```bash
mkdir /run/sshd && apt-get update && apt-get install -y openssh-server
# Start by
/usr/sbin/sshd
# Or Start as a service
/usr/sbin/sshd -D 
```
如果通过服务启动sshd,则可以通过下列命令管理sshd
```bash
# 查看状态
service ssh status
# 重启服务
service ssh restart
# 关闭服务
service ssh stop
# 启动服务
service ssh start
# 共支持方法为{start|stop|reload|force-reload|restart|try-restart|status}
```
#### Conda
通过miniforge来安装conda  
将 [Miniforge3-24.1.2-0-Linux-x86_64.sh](https://release-assets.githubusercontent.com/github-production-release-asset/221584272/c964116d-257a-427b-805d-e939abee0832?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-08-20T16%3A08%3A36Z&rscd=attachment%3B+filename%3DMiniforge3-24.1.2-0-Linux-x86_64.sh&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-08-20T15%3A08%3A35Z&ske=2025-08-20T16%3A08%3A36Z&sks=b&skv=2018-11-09&sig=4Fdh1IUJgCCxA52uwpbjYoE2vuSX25Yo5RJGVdgVisg%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc1NTcwMzM5OSwibmJmIjoxNzU1NzAzMDk5LCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.fIYeayJoWsbeEs4sqjW2dF8NhPwYaBMVBqhctGckZX4&response-content-disposition=attachment%3B%20filename%3DMiniforge3-24.1.2-0-Linux-x86_64.sh&response-content-type=application%2Foctet-stream)文件上传至容器内执行安装  
也可以在[miniforge release](https://conda-forge.org/miniforge/)上选择其他版本  
```bash
chmod +x ./Miniforge3-24.1.2-0-Linux-x86_64.sh
./Miniforge3-24.1.2-0-Linux-x86_64.sh
```
##### ~/.bashrc
建议使用南科大源作为镜像源  
因为该镜像源包含了pytorch和nvidia
```yaml
channels:
  - defaults
show_channel_urls: true
default_channels:
  - https://mirrors.sustech.edu.cn/anaconda/pkgs/main
  - https://mirrors.sustech.edu.cn/anaconda/pkgs/free
  - https://mirrors.sustech.edu.cn/anaconda/pkgs/r
  - https://mirrors.sustech.edu.cn/anaconda/pkgs/pro
  - https://mirrors.sustech.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.sustech.edu.cn/anaconda/cloud
  msys2: https://mirrors.sustech.edu.cn/anaconda/cloud
  bioconda: https://mirrors.sustech.edu.cn/anaconda/cloud
  menpo: https://mirrors.sustech.edu.cn/anaconda/cloud
  pytorch: https://mirrors.sustech.edu.cn/anaconda/cloud
  simpleitk: https://mirrors.sustech.edu.cn/anaconda/cloud
  nvidia: https://mirrors.sustech.edu.cn/anaconda-extra/cloud
```
## VLLM安装
### conda环境
容器内的python构建在根目录下，建议从conda中重新构建环境
通过```bash```命令切换进入bash环境  
执行以下命令创建并进入一个新的conda环境  
其中```${CONDA_ENV_NAME}```为您设定的conda环境名称
```bash
bash
conda create -n ${CONDA_ENV_NAME} python=3.12
conda activate ${CONDA_ENV_NAME}
```
### pytorch安装
使用最新的2.8.0版本torch
```bash
pip install \
    "numpy<2" \
    torch==2.8.0 torchvision==0.23.0 \
    --index-url https://download.pytorch.org/whl/cu129
```

### VLLM和transformer安装
请通过预构建好的vllm二进制发行包来安装
```bash
pip install \
    "numpy<2" \
    "transformers<4.54.0" \
    vllm-0.9.3.dev0+ga5dd03c1e.d20250819.cu129-cp312-cp312-linux_x86_64.whl
```

## 测试
### 推理服务
通过下列命令启动vllm的推理服务
```bash
python -m vllm.entrypoints.openai.api_server \
    --served-model-name Qwen2-7B-Instruct \
    --model /root/crr/Qwen2.5-7B-Instruct \
    --gpu-memory-utilization 0.2 \
    --port 20000
```
### 接口测试
下列命令如果没有任何返回则证明服务已正常运行
```bash
curl http://127.0.0.1:20000/health
```
下列命令可以与模型进行对话
```bash
curl http://127.0.0.1:20000/v1/chat/completions \
    --header 'Content-Type: application/json' \
    --data-raw '{
        "model": "Qwen2-7B-Instruct",
    "messages": [{
        "role": "user",
        "content": "你好"
    }],
  "stream": false
}'
```

## 编译vllm
### 环境准备
参考前文章节 ```vllm安装-conda环境-pytorch安装```
### vllm源码
#### 获取
```bash
git clone -b v0.9.2 https://github.com/vllm-project/vllm.git
```
#### 修改
##### pyproject.toml
删除9行附近的```    "torch == 2.7.0",```
##### setup.cfg
添加```setup.cfg```
```conf
[easy_install]
index_url = https://mirrors.bfsu.edu.cn/pypi/web/simple
```
##### requirements/build.txt
删除```torch==2.7.0```
##### requirements/cuda.txt
删除下列行
```
torch==2.7.0
torchaudio==2.7.0
# These must be updated alongside torch
torchvision==0.22.0 # Required for phi3v processor. See https://github.com/pytorch/vision?tab=readme-ov-file#installation for corresponding version
# https://github.com/facebookresearch/xformers/releases/tag/v0.0.30
xformers==0.0.30; platform_system == 'Linux' and platform_machine == 'x86_64'  # Requires PyTorch >= 2.7
```
#### 使用现有pytorch安装
##### *可选项 安装ccache加速c++编译
```bash
apt install -y ccache
```
或通过conda安装
```bash
conda install conda-forge::ccache
```
根据vllm[官方文档](https://vllm.hyper.ai/docs/getting-started/installation/gpu/#%E4%BD%BF%E7%94%A8%E7%8E%B0%E6%9C%89-pytorch-%E5%AE%89%E8%A3%85)  
通过下列命令修改源码文件的依赖项配置  
```bash
git clone -b v0.9.2 https://github.com/vllm-project/vllm.git
cd vllm
python use_existing_torch.py
```
开始编译，如果内存足够大,则加大MAX_JOBS来加速编译
```
pip install -r requirements/build.txt
mkdir -p ../whl
MAX_JOBS=8 python setup.py bdist_wheel -d ../whl 2>&1 | tee ../build_whl.log
```
如果之前有失败的编译记录，则需要先清除上次的构建文件夹
```
cd vllm
rm -rf build/*
```