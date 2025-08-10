---
title: "Lammps build with nequip"
date: 2025-08-09T23:02:22+08:00
draft: false
description: "使用4090运行带nequip的lammps"
tags: ["x86", "lammps", "build"]
---

1. numpy必须使用1.x
2. cmake 对于nvtx的支持存在问题,必须手动指定
```cmake
find_path(nvtx3_dir NAMES nvtx3 PATHS "/data/apps/cuda/12.6/include")
```
3. 使用conda安装mpi时,默认使用了mpich,mpich不提供`OMPI_`系列的环境变量,因此需要手动指定openmpi作为mpi后端
4. 项目地址:[pair_nequip_allegro](https://github.com/mir-group/pair_nequip_allegro)
5. 建议使用`nvcc_warpper`作为编译前端,路径在`${LAMMPS_SRC}/lib/kokkos/bin/nvcc_wrapper`
```bash
default_arch="sm_89"  
host_compiler='${CONDA_PREFIX}/bin/x86_64-conda-linux-gnu-g++'
```

{{<gist bio-punk 69d74493c6efc5c83f41d67a2c7c90f7>}}