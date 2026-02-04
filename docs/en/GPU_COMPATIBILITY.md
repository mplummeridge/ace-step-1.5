# GPU Compatibility Guide

ACE-Step 1.5 automatically adapts to your GPU's available VRAM, adjusting generation limits and LM model availability accordingly. The system detects GPU memory at startup and configures optimal settings.

## Supported GPU Backends

ACE-Step 1.5 supports multiple GPU backends:

- **NVIDIA GPUs (CUDA)**: Full support with CUDA 12.8+
- **AMD GPUs (ROCm)**: Full support with ROCm 5.7+
- **Intel Arc GPUs (XPU)**: Support via Intel Extension for PyTorch
- **Apple Silicon (MPS)**: Basic support on macOS

All GPU backends share the same tier configuration system based on available VRAM.

### AMD GPU Setup (ROCm)

For AMD GPUs, you need to install PyTorch with ROCm support:

```bash
# Install PyTorch with ROCm 6.2 support
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/rocm6.2

# Then install ACE-Step
cd ACE-Step-1.5
uv sync
```

ACE-Step will automatically detect ROCm and configure the system accordingly. ROCm GPUs use the same VRAM-based tier system as NVIDIA GPUs.

## GPU Tier Configuration

| VRAM | Tier | LM Mode | Max Duration | Max Batch Size | LM Memory Allocation |
|------|------|---------|--------------|----------------|---------------------|
| ≤4GB | Tier 1 | Not available | 3 min | 1 | - |
| 4-6GB | Tier 2 | Not available | 6 min | 1 | - |
| 6-8GB | Tier 3 | 0.6B (optional) | With LM: 4 min / Without: 6 min | With LM: 1 / Without: 2 | 3GB |
| 8-12GB | Tier 4 | 0.6B (optional) | With LM: 4 min / Without: 6 min | With LM: 2 / Without: 4 | 3GB |
| 12-16GB | Tier 5 | 0.6B / 1.7B | With LM: 4 min / Without: 6 min | With LM: 2 / Without: 4 | 0.6B: 3GB, 1.7B: 8GB |
| 16-24GB | Tier 6 | 0.6B / 1.7B / 4B | 8 min | With LM: 4 / Without: 8 | 0.6B: 3GB, 1.7B: 8GB, 4B: 12GB |
| ≥24GB | Unlimited | All models | 10 min | 8 | Unrestricted |

## Notes

- **Default settings** are automatically configured based on detected GPU memory
- **LM Mode** refers to the Language Model used for Chain-of-Thought generation and audio understanding
- **Flash Attention**, **CPU Offload**, **Compile**, and **Quantization** are enabled by default for optimal performance
- If you request a duration or batch size exceeding your GPU's limits, a warning will be displayed and values will be clamped
- **Constrained Decoding**: When LM is initialized, the LM's duration generation is also constrained to the GPU tier's maximum duration limit, preventing out-of-memory errors during CoT generation
- For GPUs with ≤6GB VRAM, LM initialization is disabled by default to preserve memory for the DiT model
- You can manually override settings via command-line arguments or the Gradio UI

> **Community Contributions Welcome**: The GPU tier configurations above are based on our testing across common hardware. If you find that your device's actual performance differs from these parameters (e.g., can handle longer durations or larger batch sizes), we welcome you to conduct more thorough testing and submit a PR to optimize these configurations in `acestep/gpu_config.py`. Your contributions help improve the experience for all users!

## Memory Optimization Tips

1. **Low VRAM (<8GB)**: Use DiT-only mode without LM initialization for maximum duration
2. **Medium VRAM (8-16GB)**: Use the 0.6B LM model for best balance of quality and memory
3. **High VRAM (>16GB)**: Enable larger LM models (1.7B/4B) for better audio understanding and generation quality

## Tested Hardware

### NVIDIA GPUs
- RTX 3090 (24GB) - Tier Unlimited
- RTX 3080 (10GB/12GB) - Tier 4/5
- RTX 4090 (24GB) - Tier Unlimited
- A100 (40GB/80GB) - Tier Unlimited

### AMD GPUs (ROCm)
AMD GPUs with ROCm 5.7+ are supported. The tier system works the same as NVIDIA GPUs based on VRAM:
- RX 7900 XTX (24GB) - Tier Unlimited
- RX 7900 XT (20GB) - Tier 6
- RX 6800 XT (16GB) - Tier 6
- RX 6700 XT (12GB) - Tier 5

> **Note for AMD GPU Users**: Make sure you have ROCm 5.7 or later installed. ACE-Step will automatically detect and use ROCm when available.

## Debug Mode: Simulating Different GPU Configurations

For testing and development, you can simulate different GPU memory sizes using the `MAX_CUDA_VRAM` environment variable:

```bash
# Simulate a 4GB GPU (Tier 1)
MAX_CUDA_VRAM=4 uv run acestep

# Simulate an 8GB GPU (Tier 4)
MAX_CUDA_VRAM=8 uv run acestep

# Simulate a 12GB GPU (Tier 5)
MAX_CUDA_VRAM=12 uv run acestep

# Simulate a 16GB GPU (Tier 6)
MAX_CUDA_VRAM=16 uv run acestep
```

This is useful for:
- Testing GPU tier configurations on high-end hardware
- Verifying that warnings and limits work correctly for each tier
- Developing and testing new GPU configuration parameters before submitting a PR
