# ComfyUI Conda 环境搭建指南

> 这是一个 [Claude Code Skill](https://docs.anthropic.com/en/docs/claude-code/skills)，用于在 Windows 平台上为 ComfyUI 快速搭建 conda 虚拟环境。

## 概述

本 Skill 提供一套完整的 ComfyUI 运行环境配置流程，涵盖从创建 conda 环境到修复已知兼容性问题的全部步骤。

## 目标环境配置

| 项目 | 配置 |
|------|------|
| 操作系统 | Windows 10 / 11 |
| GPU 驱动 | NVIDIA 显卡驱动 (≥ 560.00) |
| Conda 环境 | `cfpy312t290cu130` |
| Python | 3.12 |
| PyTorch | 2.9.0 |
| CUDA Toolkit | 13.0 |
| torchvision | 0.24.0 |
| torchaudio | 2.9.0 |
| xformers | 0.0.33 |
| SageAttention | 1.0.6+ |
| triton | 3.6.0+ (via triton-windows) |
| torchmetrics | 1.8.2 |
| torchsde | 0.2.6 |
| opencv-python | 最新版 (NumPy 2.x 兼容) |
| NumPy | 2.x (PyTorch 自动安装) |

## 适配的 NVIDIA 显卡

本环境基于 **CUDA 13.0** 构建，兼容以下 NVIDIA GPU 架构：

### 推荐显卡（完整支持，性能最佳）

| 架构 | 显卡型号 | 显存 | 说明 |
|------|---------|------|------|
| **Blackwell** | RTX 5090 | 32GB GDDR7 | 旗舰级，本环境基于此测试通过 |
| **Blackwell** | RTX 5080 | 16GB GDDR7 | 高端 |
| **Blackwell** | RTX 5070 Ti | 16GB GDDR7 | 高端 |
| **Blackwell** | RTX 5070 | 12GB GDDR7 | 中高端 |
| **Ada Lovelace** | RTX 4090 | 24GB GDDR6X | 上一代旗舰 |
| **Ada Lovelace** | RTX 4080 / 4080 SUPER | 16/16GB GDDR6X | 高端 |
| **Ada Lovelace** | RTX 4070 Ti / Ti SUPER | 12/16GB GDDR6X | 中高端 |
| **Ada Lovelace** | RTX 4070 / SUPER | 12/12GB GDDR6X | 中高端 |
| **Ada Lovelace** | RTX 4060 Ti | 8/16GB GDDR6 | 中端 |

### 兼容显卡（可用，部分特性受限）

| 架构 | 显卡型号 | 显存 | 说明 |
|------|---------|------|------|
| **Ampere** | RTX 3090 / 3090 Ti | 24GB GDDR6X | 8GB+ 显存运行流畅 |
| **Ampere** | RTX 3080 / 3080 Ti | 10/12GB GDDR6X | 10GB 起步可用 |
| **Ampere** | RTX 3070 / Ti | 8GB GDDR6 | 8GB 为推荐最低显存 |
| **Ampere** | RTX 3060 Ti | 8GB GDDR6 | 勉强可用，需小 batch size |
| **Ampere** | RTX 3060 | 12GB GDDR6 | 显存够但算力有限 |
| **Turing** | RTX 2080 Ti | 11GB GDDR6 | 老将仍能战 |
| **Turing** | RTX 2080 / 2070 SUPER | 8GB GDDR6 | 最低推荐线 |

### 不推荐的显卡

| 架构 | 显卡型号 | 原因 |
|------|---------|------|
| **Turing** | RTX 2060 (6GB) | 显存不足，大多数模型无法加载 |
| **Pascal** | GTX 1080 Ti (11GB) | 架构过旧，算力不足，缺少新特性 |
| **Pascal** | GTX 1070 / 1060 | 显存与算力双重不足 |
| **Maxwell 及更早** | GTX 900 系列及以下 | 不支持 CUDA 13.0 所需的算力特性 |

> **显存建议：** 运行 SDXL/Flux 等大模型建议 12GB+ 显存，运行 SD 1.5 等小模型 8GB 起步。显存不足时可启用 `--lowvram` 模式，但会显著降低速度。

## 快速开始

### 前提条件

- Windows 10/11
- 已安装 Miniconda 或 Anaconda
- NVIDIA 显卡（见上方兼容列表）
- 最新 NVIDIA 驱动（建议 ≥ 560.00）

### 一键安装

```bash
# 1. 创建 conda 环境
conda create -n cfpy312t290cu130 python=3.12 -y

# 2. 安装 PyTorch (CUDA 13.0)
conda run -n cfpy312t290cu130 pip install torch==2.9.0 torchvision==0.24.0 torchaudio==2.9.0 --index-url https://download.pytorch.org/whl/cu130

# 3. 安装 torchmetrics 和 torchsde
conda run -n cfpy312t290cu130 pip install torchmetrics==1.8.2 torchsde==0.2.6

# 4. 安装 xformers、SageAttention、triton
conda run -n cfpy312t290cu130 pip install xformers==0.0.33
conda run -n cfpy312t290cu130 pip install sageattention
conda run -n cfpy312t290cu130 pip install triton-windows

# 5. 修复 opencv-python 与 NumPy 2.x 的兼容性
conda run -n cfpy312t290cu130 pip install --force-reinstall --no-cache-dir opencv-python
```

### 验证安装

```bash
conda run -n cfpy312t290cu130 python -c "
import torch, torchvision, torchaudio, torchmetrics, torchsde
import xformers, sageattention, triton, cv2, numpy as np
print(f'PyTorch {torch.__version__} | CUDA {torch.version.cuda} | NumPy {np.__version__}')
print(f'GPU: {torch.cuda.get_device_name(0)}')
print('All packages OK!')
"
```

## 已知问题与解决方案

### triton 不支持 Windows

官方 `triton` 包没有 Windows 版本，安装会报错。使用社区维护的 `triton-windows` 替代，导入时仍为 `import triton`，对 ComfyUI 完全透明。

### opencv-python 与 NumPy 2.x 不兼容

PyTorch 2.9.0 会安装 NumPy 2.x，而旧版 opencv-python 与之 ABI 不兼容，导致：

```
AttributeError: _ARRAY_API not found
ImportError: numpy.core.multiarray failed to import
```

所有使用 `cv2` 的 ComfyUI 自定义节点会同时崩溃。解决方法：

```bash
pip install --force-reinstall --no-cache-dir opencv-python
```

ComfyUI 的 WAS Node Suite 会尝试自动修复此问题，但在 Windows 上可能因跨盘符文件操作失败（`WinError 17`、`WinError 5`）而失败，因此需要手动执行。

## 安装为 Claude Code Skill

将 `SKILL.md` 复制到 Claude Code 的 skills 目录即可：

```bash
mkdir -p ~/.claude/skills/comfyui-conda-setup
cp SKILL.md ~/.claude/skills/comfyui-conda-setup/
```

安装后，在 Claude Code 中提及"ComfyUI 环境搭建"、"conda 配置"、"PyTorch CUDA" 等关键词即可自动触发此 Skill。

## 测试环境

本环境在以下硬件上完成全流程测试：

- **GPU:** NVIDIA GeForce RTX 5090 (32GB GDDR7 VRAM)
- **内存:** 65GB DDR5
- **操作系统:** Windows 11 Pro
- **ComfyUI 路径:** `G:\ComfyUI`

## License

MIT
