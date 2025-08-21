---
title: "Lammps_with_mace"
date: 2025-08-20T21:43:50+08:00
draft: false
description: ""
tags: ["lammps", "build"]
---

# LAMMPS with mace

## 资源
4090

## 环境创建
这个比较麻烦，torch不支持torch2.6，最大只能支持到2.4.0  
因此pytorch里面没有带cxx11-abi，必须使用libtorch  
libtorch选择```https://download.pytorch.org/libtorch/cu124/libtorch-shared-with-deps-2.4.0%2Bcu124.zip```  
同时源码文件使用```git clone --branch=mace --depth=1 https://github.com/ACEsuit/lammps ${LAMMPS_SRC}```
同时kokkos_arch中没有爱达·拉芙蕾丝,因此通过GPU_ARCH=sm_89 决定架构
构建环境使用```env_create.sh```

## 编译
### 编译lammps
```sbatch build_lammps.sh```
### 编译mace
```sbatch build_mace.sh```

## 测试
```sbatch run.sh```

{{<gist bio-punk ab4cf4e2b1ee15a2d86ad764ffe808aa>}}