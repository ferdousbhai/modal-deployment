# Images Reference

Modal Images define the container environment where your code runs.

## Table of Contents
- [Base Images](#base-images)
- [Installing Packages](#installing-packages)
- [Adding Local Files](#adding-local-files)
- [Running Commands](#running-commands)
- [Using Existing Images](#using-existing-images)
- [Build-time Functions](#build-time-functions)
- [Caching and Rebuilds](#caching-and-rebuilds)

## Base Images

### debian_slim (Recommended)
```python
image = modal.Image.debian_slim(python_version="3.12")
```

### micromamba (Conda packages)
```python
image = modal.Image.micromamba().micromamba_install(
    "pytorch", "cudatoolkit=11.8",
    channels=["pytorch", "conda-forge"]
)
```

## Installing Packages

### Python Packages
```python
# Using uv (faster, recommended)
image = modal.Image.debian_slim().uv_pip_install(
    "torch==2.3.0",
    "transformers>=4.40.0",
    "numpy",
)

# Using pip
image = modal.Image.debian_slim().pip_install("pandas", "scikit-learn")

# From requirements.txt
image = modal.Image.debian_slim().pip_install_from_requirements("requirements.txt")

# Private repos
image = modal.Image.debian_slim().pip_install_private_repos(
    "github.com/org/private-repo@main",
    git_user="username",
    secrets=[modal.Secret.from_name("github-token")]
)
```

### System Packages
```python
image = modal.Image.debian_slim().apt_install(
    "ffmpeg", "libsndfile1", "git", "curl"
)
```

### GPU during build
Some packages need GPU access during installation:
```python
image = modal.Image.debian_slim().pip_install("bitsandbytes", gpu="H100")
```

## Adding Local Files

### Python Modules
```python
# Add importable module
image = modal.Image.debian_slim().add_local_python_source("my_package")
```

### Directories and Files
```python
image = (
    modal.Image.debian_slim()
    .add_local_dir("./data", remote_path="/app/data")
    .add_local_file("config.yaml", remote_path="/app/config.yaml")
)
```

Use `copy=True` to include files in the image layer (enables subsequent build steps):
```python
image = modal.Image.debian_slim().add_local_dir("./src", remote_path="/src", copy=True)
```

## Running Commands

```python
image = (
    modal.Image.debian_slim()
    .run_commands(
        "git clone https://github.com/org/repo /opt/repo",
        "cd /opt/repo && pip install -e .",
    )
    .workdir("/opt/repo")
)
```

## Using Existing Images

### From Registry
```python
# Public registry
image = modal.Image.from_registry("nvidia/cuda:12.4.0-devel-ubuntu22.04", add_python="3.11")

# Docker Hub
image = modal.Image.from_registry("pytorch/pytorch:2.3.0-cuda12.1-cudnn8-runtime")
```

### Private Registries

**Docker Hub:**
```python
image = modal.Image.from_registry(
    "myorg/private-image:latest",
    secret=modal.Secret.from_name("dockerhub-secret")  # REGISTRY_USERNAME, REGISTRY_PASSWORD
)
```

**AWS ECR:**
```python
image = modal.Image.from_aws_ecr(
    "123456789.dkr.ecr.us-east-1.amazonaws.com/my-image:latest",
    secret=modal.Secret.from_name("aws-secret")
)
```

**GCP Artifact Registry:**
```python
image = modal.Image.from_gcp_artifact_registry(
    "us-docker.pkg.dev/project/repo/image:tag",
    secret=modal.Secret.from_name("gcp-secret")
)
```

### From Dockerfile
```python
image = modal.Image.from_dockerfile("./Dockerfile")
```

## Build-time Functions

Run Python code during image build:

```python
def download_models():
    from huggingface_hub import snapshot_download
    snapshot_download("meta-llama/Llama-3-8B", local_dir="/models")

image = (
    modal.Image.debian_slim()
    .pip_install("huggingface-hub")
    .run_function(
        download_models,
        secrets=[modal.Secret.from_name("hf-token")],
        gpu="A10G",  # If GPU needed during download
    )
)
```

## Environment Variables

```python
image = modal.Image.debian_slim().env({
    "HF_HOME": "/cache/huggingface",
    "TRANSFORMERS_CACHE": "/cache/transformers",
    "CUDA_VISIBLE_DEVICES": "0",
})
```

## Handling Remote-only Imports

When packages aren't installed locally:

```python
# Option 1: Import inside function
@app.function(image=torch_image)
def train():
    import torch  # Only imported in container
    return torch.cuda.is_available()

# Option 2: Use imports context manager
with torch_image.imports():
    import torch
    import transformers

@app.function(image=torch_image)
def train():
    return torch.cuda.is_available()
```

## Caching and Rebuilds

Images are cached per layer. Breaking cache on one layer cascades to subsequent layers.

**Force rebuild:**
```python
image = (
    modal.Image.debian_slim()
    .apt_install("git")
    .pip_install("transformers", force_build=True)  # This and subsequent layers rebuild
)
```

**Environment variables:**
- `MODAL_FORCE_BUILD=1` - Rebuild all images
- `MODAL_IGNORE_CACHE=1` - Rebuild without breaking cache

**Best Practice:** Put frequently-changing layers last:
```python
image = (
    modal.Image.debian_slim()
    .apt_install("system-deps")           # Rarely changes
    .pip_install("torch", "transformers") # Occasionally changes
    .add_local_python_source("my_code")   # Changes frequently (last)
)
```
