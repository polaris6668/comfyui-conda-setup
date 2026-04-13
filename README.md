# ComfyUI Conda 环境搭建指南

> 这是一个 [Claude Code Skill](https://docs.anthropic.com/en/docs/claude-code/skills)，用于在 Windows 平台上为 ComfyUI 快速搭建 conda 虚拟环境。

## 概述

本 Skill 提供一套完整的 ComfyUI 运行环境配置流程，涵盖从创建 conda 环境到修复已知兼容性问题的全部步骤。

## 环境版本

| 组件 | 版本 |
|------|------|
| Python | 3.12 |
| PyTorch | 2.9.0 |
| torchvision | 0.24.0 |
| torchaudio | 2.9.0 |
| CUDA | 13.0 |
| xformers | 0.0.33 |
| SageAttention | 1.0.6+ |
| triton-windows | 3.6.0+ |
| torchmetrics | 1.8.2 |
| torchsde | 0.2.6 |

## 快速开始

### 前提条件

- Windows 10/11
- 已安装 Miniconda 或 Anaconda
- NVIDIA 显卡

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

## 参考硬件配置

- GPU: NVIDIA GeForce RTX 5090 (32GB VRAM)
- RAM: 65GB
- OS: Windows 11 Pro

此配置适用于大多数 NVIDIA RTX 系列 GPU，根据显存调整 ComfyUI 的 batch size 即可。

## License

MIT
