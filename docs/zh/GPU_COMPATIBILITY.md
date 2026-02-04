# GPU 兼容性指南

ACE-Step 1.5 会自动适配您的 GPU 显存大小，相应调整生成时长限制和可用的 LM 模型。系统在启动时检测 GPU 显存并自动配置最佳设置。

## 支持的 GPU 后端

ACE-Step 1.5 支持多种 GPU 后端：

- **NVIDIA GPU (CUDA)**: 完全支持 CUDA 12.8+
- **AMD GPU (ROCm)**: 完全支持 ROCm 5.7+
- **Intel Arc GPU (XPU)**: 通过 Intel Extension for PyTorch 支持
- **Apple Silicon (MPS)**: 在 macOS 上基本支持

所有 GPU 后端共享基于可用显存的分级配置系统。

### AMD GPU 设置 (ROCm)

对于 AMD GPU，您需要安装支持 ROCm 的 PyTorch：

```bash
# 安装支持 ROCm 6.2 的 PyTorch
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/rocm6.2

# 然后安装 ACE-Step
cd ACE-Step-1.5
uv sync
```

ACE-Step 将自动检测 ROCm 并相应配置系统。ROCm GPU 使用与 NVIDIA GPU 相同的基于显存的分级系统。

## GPU 分级配置

| 显存 | 等级 | LM 模式 | 最大时长 | 最大批次 | LM 显存分配 |
|------|------|---------|----------|----------|-------------|
| ≤4GB | Tier 1 | 不可用 | 3 分钟 | 1 | - |
| 4-6GB | Tier 2 | 不可用 | 6 分钟 | 1 | - |
| 6-8GB | Tier 3 | 0.6B (可选) | 有 LM: 4 分钟 / 无 LM: 6 分钟 | 有 LM: 1 / 无 LM: 2 | 3GB |
| 8-12GB | Tier 4 | 0.6B (可选) | 有 LM: 4 分钟 / 无 LM: 6 分钟 | 有 LM: 2 / 无 LM: 4 | 3GB |
| 12-16GB | Tier 5 | 0.6B / 1.7B | 有 LM: 4 分钟 / 无 LM: 6 分钟 | 有 LM: 2 / 无 LM: 4 | 0.6B: 3GB, 1.7B: 8GB |
| 16-24GB | Tier 6 | 0.6B / 1.7B / 4B | 8 分钟 | 有 LM: 4 / 无 LM: 8 | 0.6B: 3GB, 1.7B: 8GB, 4B: 12GB |
| ≥24GB | 无限制 | 所有模型 | 10 分钟 | 8 | 无限制 |

## 说明

- **默认设置** 会根据检测到的 GPU 显存自动配置
- **LM 模式** 指用于思维链 (Chain-of-Thought) 生成和音频理解的语言模型
- **Flash Attention**、**CPU Offload**、**Compile** 和 **Quantization** 默认启用以获得最佳性能
- 如果您请求的时长或批次大小超出 GPU 限制，系统会显示警告并自动调整到允许的最大值
- **约束解码**: 当 LM 初始化后，LM 生成的时长也会被约束在 GPU 等级的最大时长限制内，防止在 CoT 生成时出现显存不足错误
- 对于显存 ≤6GB 的 GPU，默认禁用 LM 初始化以保留显存给 DiT 模型
- 您可以通过命令行参数或 Gradio UI 手动覆盖设置

> **欢迎社区贡献**: 以上 GPU 分级配置基于我们在常见硬件上的测试。如果您发现您的设备实际性能与这些参数不符（例如，可以处理更长的时长或更大的批次），欢迎您进行更充分的测试，并提交 PR 来优化 `acestep/gpu_config.py` 中的配置。您的贡献将帮助改善所有用户的体验！

## 显存优化建议

1. **低显存 (<8GB)**: 使用纯 DiT 模式，不初始化 LM，以获得最大时长
2. **中等显存 (8-16GB)**: 使用 0.6B LM 模型，在质量和显存之间取得最佳平衡
3. **高显存 (>16GB)**: 启用更大的 LM 模型 (1.7B/4B) 以获得更好的音频理解和生成质量

## 已测试硬件

### NVIDIA GPU
- RTX 3090 (24GB) - Tier 无限制
- RTX 3080 (10GB/12GB) - Tier 4/5
- RTX 4090 (24GB) - Tier 无限制
- A100 (40GB/80GB) - Tier 无限制

### AMD GPU (ROCm)
支持 ROCm 5.7+ 的 AMD GPU。分级系统与 NVIDIA GPU 相同，基于显存：
- RX 7900 XTX (24GB) - Tier 无限制
- RX 7900 XT (20GB) - Tier 6
- RX 6800 XT (16GB) - Tier 6
- RX 6700 XT (12GB) - Tier 5

> **AMD GPU 用户注意**: 请确保安装 ROCm 5.7 或更高版本。ACE-Step 会在可用时自动检测并使用 ROCm。

## 调试模式：模拟不同的 GPU 配置

在测试和开发时，您可以使用 `MAX_CUDA_VRAM` 环境变量来模拟不同的 GPU 显存大小：

```bash
# 模拟 4GB GPU (Tier 1)
MAX_CUDA_VRAM=4 uv run acestep

# 模拟 8GB GPU (Tier 4)
MAX_CUDA_VRAM=8 uv run acestep

# 模拟 12GB GPU (Tier 5)
MAX_CUDA_VRAM=12 uv run acestep

# 模拟 16GB GPU (Tier 6)
MAX_CUDA_VRAM=16 uv run acestep
```

适用场景：
- 在高端硬件上测试 GPU 分级配置
- 验证各等级的警告和限制是否正常工作
- 在提交 PR 之前开发和测试新的 GPU 配置参数
