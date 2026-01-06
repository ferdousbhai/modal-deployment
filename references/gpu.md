# GPU Reference

## Table of Contents
- [GPU Types](#gpu-types)
- [Specifying GPUs](#specifying-gpus)
- [Multi-GPU](#multi-gpu)
- [GPU Fallbacks](#gpu-fallbacks)
- [CUDA Setup](#cuda-setup)
- [Best Practices](#best-practices)

## GPU Types

| GPU | VRAM | Best For | Price Tier |
|-----|------|----------|------------|
| B200 | 192GB | Largest models, latest architecture | $$$$$ |
| H200 | 141GB | Large LLMs, high bandwidth | $$$$ |
| H100 | 80GB | LLM inference/training, transformers | $$$$ |
| A100-80GB | 80GB | Large model training/inference | $$$ |
| A100-40GB | 40GB | Medium-large models | $$$ |
| L40S | 48GB | Inference, video generation | $$ |
| L4 | 24GB | Cost-effective inference | $ |
| A10G | 24GB | General ML, Stable Diffusion | $ |
| T4 | 16GB | Light inference, prototyping | $ |

## Specifying GPUs

### String Syntax (Recommended)
```python
@app.function(gpu="H100")          # Single H100
@app.function(gpu="A100-80GB")     # 80GB variant
@app.function(gpu="H100:2")        # 2x H100
@app.function(gpu="A100-40GB:4")   # 4x A100-40GB
```

## Multi-GPU

### Up to 8 GPUs
```python
@app.function(gpu="H100:8")  # 640GB total VRAM
def train_405b():
    import torch
    print(f"GPUs available: {torch.cuda.device_count()}")
```

Supported multi-GPU counts:
- B200, H200, H100, A100, L4, T4, L40S: up to 8
- A10G: up to 4

### Multi-GPU Training Patterns

**PyTorch DDP:**
```python
@app.function(gpu="H100:4")
def train_distributed():
    import torch.distributed as dist
    dist.init_process_group(backend="nccl")
    # Training code...
```

**Using subprocess for frameworks like Lightning:**
```python
@app.function(gpu="H100:4")
def train():
    import subprocess
    subprocess.run(["python", "train.py"], check=True)
```

## GPU Fallbacks

Specify fallback options in priority order:

```python
@app.function(gpu=["H100", "A100-80GB", "A100-40GB:2"])
def flexible_inference():
    """Tries H100 first, then A100-80GB, then 2x A100-40GB"""
    pass
```

Use case: Ensures you get ~80GB VRAM regardless of availability.

## Automatic Upgrades

Modal may automatically upgrade H100 requests to H200 at no extra cost:

```python
@app.function(gpu="H100")  # May run on H200
def inference():
    pass

@app.function(gpu="H100!")  # Force H100 only (no upgrade)
def benchmark():
    pass
```

## CUDA Setup

### Verify CUDA
```python
@app.function(gpu="L40S")
def check_cuda():
    import torch
    print(f"CUDA available: {torch.cuda.is_available()}")
    print(f"Device: {torch.cuda.get_device_name(0)}")
    print(f"VRAM: {torch.cuda.get_device_properties(0).total_memory / 1e9:.1f} GB")
```

### CUDA in Image Build
```python
# Some packages need CUDA during installation
image = (
    modal.Image.debian_slim()
    .pip_install("flash-attn", gpu="A100-40GB")  # Compiles CUDA kernels
)
```

### Using NVIDIA Base Images
```python
image = modal.Image.from_registry(
    "nvidia/cuda:12.4.0-devel-ubuntu22.04",
    add_python="3.11"
).pip_install("torch", "transformers")
```

## Memory Management

### Check VRAM Usage
```python
@app.function(gpu="H100")
def monitor_memory():
    import torch
    allocated = torch.cuda.memory_allocated() / 1e9
    reserved = torch.cuda.memory_reserved() / 1e9
    print(f"Allocated: {allocated:.2f} GB, Reserved: {reserved:.2f} GB")
```

### Clear CUDA Cache
```python
@app.function(gpu="H100")
def with_cleanup():
    import torch
    try:
        result = run_inference()
    finally:
        torch.cuda.empty_cache()
    return result
```

## Best Practices

### 1. Start with smaller GPUs
Test on T4/L4/A10G before scaling to H100:
```python
# Development
@app.function(gpu="T4")
def dev_inference(): pass

# Production
@app.function(gpu="H100")
def prod_inference(): pass
```

### 2. Use appropriate GPU for model size
| Model Size | Recommended GPU |
|------------|-----------------|
| <7B params | L4, A10G, T4 |
| 7B-13B | L40S, A100-40GB |
| 13B-70B | A100-80GB, H100 |
| 70B-405B | H100:4-8, H200:4-8 |

### 3. Optimize cold starts with snapshots
```python
@app.cls(
    gpu="H100",
    enable_memory_snapshot=True,
    experimental_options={"enable_gpu_snapshot": True}
)
class FastModel:
    @modal.enter(snap=True)
    def load(self):
        self.model = load_to_gpu()
```

### 4. Use GPU fallbacks for reliability
```python
@app.function(gpu=["H100", "A100-80GB"])
def reliable_inference():
    pass
```

### 5. Reserve CPU/memory alongside GPU
```python
@app.function(gpu="H100", cpu=8, memory=65536)
def training():
    # 8 CPU cores, 64GB RAM, 1 H100
    pass
```
