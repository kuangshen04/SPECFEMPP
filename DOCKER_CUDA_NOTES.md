# SPECFEM++ CUDA (sm_120) · Docker 使用说明

构建于 2026-10-05，目标机器：RTX 5070 Ti Laptop (CC 12.0 / sm_120)

## 镜像

| 镜像 | 大小 | 用途 |
|---|---|---|
| `specfempp-cuda:0.6.0` | 840 MB | **日常运行**。只含可执行文件 + 运行期依赖 |
| `specfempp-cuda:builder` | 12.8 GB | 改代码后**重新编译**用。含 nvcc 13.2 / GCC 13.3 / gfortran / CMake 及 Kokkos+Boost 构建树 |

可执行文件位于镜像内 `/opt/specfempp/bin/release-cuda/`：
`specfem`、`xmeshfem2D`、`xmeshfem3D`、`xdecompose_mesh`、`xadj_seismogram`

## 运行算例

```powershell
# 2D 例子（注意必须带 --gpus all）
docker run --rm --gpus all -it `
  -v "C:/Users/KuangShen/Desktop/SPECFEMPP/examples/dim2/homogeneous-elastic:/work" `
  -w /work specfempp-cuda:0.6.0 bash -lc "
    mkdir -p OUTPUT_FILES/results &&
    xmeshfem2D -p Par_File &&
    specfem 2d -p specfem_config.yaml
  "
```

## ⚠️ 头号坑：Windows 的 CRLF 行尾

Fortran 网格器把 `topography.dat\r` 当成文件名，直接报
`Error opening file: topography.dat` 而失败。

**在容器里跑之前，先把文本输入文件的 CRLF 去掉：**

```bash
for f in Par_File topography.dat specfem_config.yaml source.yaml STATIONS; do
  tr -d '\r' < "$f" > "$f.lf" && mv "$f.lf" "$f"
done
```

（`specfem_config.yaml` 由 yaml-cpp 读，不受影响；但 `Par_File` /
`topography.dat` / `STATIONS` 等 Fortran 侧文件必须转成 LF。）

## 验证结果

在 `tests/integration-tests/.../HomogeneousElasticDomain` 用例上，
按官方测试口径（每台站各分量合并后取 error/norm）与 Fortran 原版参考道对比：

* **20/20 台站 PASS**
* 最大误差比 **1.87e-4**（官方容差 1e-3）
* GPU 确认在用：`/proc/<pid>/maps` 有 20 条 nvidia 映射，
  吞吐约 580 步/秒（80×60 网格）
* 注：SPECFEM++ 默认单精度（`SPECFEM_ENABLE_DOUBLE_PRECISION=OFF`），
  与双精度参考道对比时，时间列会出现 ~1e-8 量级的舍入差，属正常。

## 改代码后重新编译

```powershell
# 1) 起一个带 GPU 的 builder 容器，把源码挂进去
docker run -d --name sfpp-build --gpus all `
  -v "C:/Users/KuangShen/Desktop/SPECFEMPP:/src:ro" `
  specfempp-cuda:builder sleep infinity

# 2) 进容器增量编译（Kokkos/Boost 已编好，只重编改动的部分）
docker exec -it sfpp-build bash -lc "
  cp -a /src/core /src/fortran /src/cmake /src/CMakeLists.txt /src/CMakePresets.json /opt/specfempp/ &&
  cd /opt/specfempp && cmake --build --preset release-cuda -j 8
"

# 3) 完事后提交成新镜像
docker commit sfpp-build specfempp-cuda:mybuild
```

首次全量编译耗时约 1.5 小时（其中 `solver/time_marching.cpp`
单个 TU 就要 20 分钟），**建议保留 builder 镜像做增量编译**。

## 从零重建

`Dockerfile.cuda`（仓库根目录）是多阶段构建文件，可完全复现：

```powershell
docker build -f Dockerfile.cuda --target builder -t specfempp-cuda:builder .
docker build -f Dockerfile.cuda -t specfempp-cuda:0.6.0 .
```

关键 CMake 参数：

```
-D Kokkos_ARCH_BLACKWELL120=ON     # GPU 架构 = CC 12.0
-D Kokkos_ENABLE_SERIAL=ON
-D SPECFEMPP_UNITY_BUILD_BATCH_SIZE=4   # 降低内存峰值
```

注意：Ubuntu 24.04 自带 Boost 1.83 < 项目要求的 1.85，
构建时会自动从 GitHub 拉 Boost 1.87 源码编译（官方支持的路径）。

## WSL2 配置

`~/.wslconfig`（原文件已备份为 `.wslconfig.bak`）：

```ini
[wsl2]
networkingMode=Mirrored
memory=14GB
processors=16
autoMemoryReclaim=gradual
```

`memory` 不要给太大：之前设 20GB 时编译把宿主机内存榨干到只剩 1.8GB，
导致 Docker 引擎返回 500 错误。
