---
name: comfyui-conda-setup
description: 在 Windows 上为 ComfyUI 创建和配置 conda 环境的完整指南。当用户提到 ComfyUI 环境搭建、conda 环境创建、PyTorch 安装、CUDA 配置、ComfyUI 依赖安装、或者需要从头开始配置 ComfyUI 运行环境时，都应使用此 skill。即使用户只是提到"装 ComfyUI"、"配环境"、"PyTorch CUDA" 等关键词，也应触发此 skill。
---

# ComfyUI Conda 环境搭建指南

本 skill 指导在 Windows 平台上为 ComfyUI 创建一个完整的 conda 虚拟环境，包括 PyTorch (CUDA)、常用依赖包的安装，以及已知的兼容性问题的修复。

## 前提条件

- Windows 10/11
- 已安装 Miniconda 或 Anaconda
- NVIDIA 显卡（本指南基于 RTX 5090, 32GB VRAM 测试通过）
- ComfyUI 已安装（如 `G:\ComfyUI`）

## 步骤总览

整个过程分为以下步骤：

1. 创建 conda 环境
2. 安装 PyTorch (CUDA 13.0)
3. 安装 torchmetrics 和 torchsde
4. 安装 xformers、SageAttention 和 triton
5. 修复 opencv-python 的 NumPy 2.x 兼容性问题

---

## 第一步：创建 conda 环境

创建一个名为 `cfpy312t290cu130` 的环境，使用 Python 3.12：

```bash
conda create -n cfpy312t290cu130 python=3.12 -y
```

环境命名规则为 `cf` + `py` + Python 版本 + `t` + PyTorch 版本 + `cu` + CUDA 版本，方便日后识别。

创建完成后激活环境：

```bash
conda activate cfpy312t290cu130
```

> 如果使用 `conda run` 方式执行命令（不需要先激活），则所有后续命令使用 `conda run -n cfpy312t290cu130` 前缀。

---

## 第二步：安装 PyTorch (CUDA 13.0)

使用 PyTorch 官方 CUDA 13.0 索引安装：

```bash
conda run -n cfpy312t290cu130 pip install torch==2.9.0 torchvision==0.24.0 torchaudio==2.9.0 --index-url https://download.pytorch.org/whl/cu130
```

**版本对应关系：**

| 组件 | 版本 |
|------|------|
| PyTorch | 2.9.0 |
| torchvision | 0.24.0 |
| torchaudio | 2.9.0 |
| CUDA | 13.0 |
| Python | 3.12 |

安装完成后验证：

```bash
conda run -n cfpy312t290cu130 python -c "import torch; print(f'PyTorch: {torch.__version__}'); print(f'CUDA available: {torch.cuda.is_available()}'); print(f'CUDA version: {torch.version.cuda}')"
```

预期输出应显示 `PyTorch: 2.9.0`，`CUDA available: True`。

---

## 第三步：安装 torchmetrics 和 torchsde

ComfyUI 的部分节点需要这两个包：

```bash
conda run -n cfpy312t290cu130 pip install torchmetrics==1.8.2 torchsde==0.2.6
```

---

## 第四步：安装 xformers、SageAttention 和 triton

逐个安装，避免依赖冲突：

```bash
# 安装 xformers（注意力机制优化）
conda run -n cfpy312t290cu130 pip install xformers==0.0.33

# 安装 SageAttention（高效注意力实现）
conda run -n cfpy312t290cu130 pip install sageattention

# 安装 triton（Windows 平台特殊处理）
conda run -n cfpy312t290cu130 pip install triton-windows
```

### Windows 平台 triton 的注意事项

**重要：** 官方 `triton` 包不支持 Windows。`pip install triton` 会报错：

```
ERROR: Could not find a version that satisfies the requirement triton
```

必须使用社区维护的 `triton-windows` 包替代。即使指定了 `--index-url https://download.pytorch.org/whl/cu130` 也找不到，因为 PyTorch 的索引同样不提供 Windows 版 triton。

`triton-windows` 会安装为 `triton` 模块，导入时仍然使用 `import triton`，对 ComfyUI 完全透明。

### SageAttention 版本验证

SageAttention 没有 `__version__` 属性，验证时只需确认能成功导入即可：

```bash
conda run -n cfpy312t290cu130 python -c "import sageattention; print('SageAttention imported successfully')"
```

---

## 第五步：安装 opencv-contrib-python（需先关闭 ComfyUI）

### 为什么用 opencv-contrib-python

部分 ComfyUI 插件（如 ComfyUI_LayerStyle）依赖 `cv2.ximgproc.guidedFilter`，该模块仅包含在 `opencv-contrib-python` 中，普通的 `opencv-python` 不提供。`opencv-contrib-python` 是 `opencv-python` 的超集，功能完全兼容。

### 兼容版本

经测试，以下组合在本环境中兼容：

| 包 | 版本 | 说明 |
|---|---|---|
| opencv-contrib-python | 4.13.0.92 | 与 NumPy 2.4.4 + PyTorch 2.9.0 兼容，含 ximgproc 模块 |
| NumPy | 2.4.4 | opencv-contrib-python 4.13.0.92 的依赖，自动安装 |

### 安装命令

```bash
# 先卸载所有其他 opencv 变体（消除冲突）
conda run -n cfpy312t290cu130 pip uninstall opencv-python opencv-python-headless opencv-contrib-python-headless -y

# 安装 opencv-contrib-python
conda run -n cfpy312t290cu130 pip install --no-cache-dir opencv-contrib-python
```

> **必须先关闭 ComfyUI！** 否则 `cv2.pyd` 被进程锁定，pip 无法覆盖文件（`WinError 5 拒绝访问`）。

### 常见问题

#### 多个 opencv 包冲突

某些 ComfyUI 节点会自动安装 `opencv-python`、`opencv-python-headless` 等变体。多个 opencv 包共存会导致 `cv2` 模块加载异常（如 `guidedFilter` 无法导入、`__version__` 缺失、`module 'cv2' has no attribute ...`）。**只保留 `opencv-contrib-python` 一个包：**

```bash
conda run -n cfpy312t290cu130 pip uninstall opencv-python opencv-python-headless opencv-contrib-python-headless -y
conda run -n cfpy312t290cu130 pip install --no-cache-dir opencv-contrib-python
```

如果卸载也报错（`WinError 17 跨盘符`），先关闭所有 Python 进程再试。

#### guidedFilter 缺失

启动时报错 `Cannot import name 'guidedFilter' from 'cv2.ximgproc'` 说明安装了不含 contrib 模块的 opencv 变体。按上述步骤重装 `opencv-contrib-python` 即可。

#### 跨盘符 WinError 17 / WinError 5

pip 在 Windows 上卸载包时会尝试将 `.pyd` 文件移动到 C 盘临时目录，如果 conda 环境在 G 盘（不同驱动器），会导致 `os.rename` 失败。解决方法：
1. 关闭所有占用该包的进程（ComfyUI 等）
2. 使用 `--force-reinstall` 覆盖安装而非先卸载
3. 如果仍有残留，手动删除 `site-packages/cv2/` 目录后重装

#### NumPy 2.x ABI 不兼容

旧版 opencv（如 4.10.0）与 NumPy 2.x 的 ABI 不兼容，报错：
```
AttributeError: _ARRAY_API not found
ImportError: numpy.core.multiarray failed to import
```
安装 `opencv-contrib-python >= 4.13.0` 即可解决。

---

## 验证清单

安装全部完成后，运行以下命令确认所有关键包正常工作：

```bash
conda run -n cfpy312t290cu130 python -c "
import torch
import torchvision
import torchaudio
import torchmetrics
import torchsde
import xformers
import sageattention
import triton
import cv2
import numpy as np

print(f'PyTorch: {torch.__version__}')
print(f'torchvision: {torchvision.__version__}')
print(f'torchaudio: {torchaudio.__version__}')
print(f'torchmetrics: {torchmetrics.__version__}')
print(f'torchsde: {torchsde.__version__}')
print(f'xformers: {xformers.__version__}')
print(f'SageAttention: OK')
print(f'triton: {triton.__version__}')
print(f'opencv: {cv2.__version__}')
print(f'NumPy: {np.__version__}')
print(f'CUDA available: {torch.cuda.is_available()}')
if torch.cuda.is_available():
    print(f'GPU: {torch.cuda.get_device_name(0)}')
print('All packages imported successfully!')
"
```

如果所有导入无报错且输出正确版本号，环境配置完成。

---

## 已安装包版本汇总

| 包 | 版本 |
|---|---|
| Python | 3.12.x |
| PyTorch | 2.9.0 |
| torchvision | 0.24.0 |
| torchaudio | 2.9.0 |
| torchmetrics | 1.8.2 |
| torchsde | 0.2.6 |
| xformers | 0.0.33 |
| SageAttention | 最新版 (1.0.6+) |
| triton-windows | 最新版 (3.6.0+) |
| opencv-contrib-python | 4.13.0.92（NumPy 2.x 兼容，含 ximgproc） |
| NumPy | 2.4.4 |

## 硬件参考配置

- GPU: NVIDIA GeForce RTX 5090 (32GB VRAM)
- RAM: 65GB
- OS: Windows 11 Pro
- ComfyUI 路径: `G:\ComfyUI`

此配置适用于大多数 NVIDIA RTX 系列 GPU，只需根据显存调整 ComfyUI 的 batch size 即可。
