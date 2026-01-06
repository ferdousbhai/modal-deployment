# Storage Reference

## Table of Contents
- [Volumes](#volumes)
- [Secrets](#secrets)
- [Cloud Bucket Mounts](#cloud-bucket-mounts)
- [Passing Local Data](#passing-local-data)
- [Storage Comparison](#storage-comparison)

## Volumes

Distributed persistent storage for model weights, datasets, and outputs.

### Create and Use

```python
volume = modal.Volume.from_name("my-volume", create_if_missing=True)

@app.function(volumes={"/data": volume})
def write_data():
    with open("/data/output.txt", "w") as f:
        f.write("Hello!")
    volume.commit()  # Persist changes

@app.function(volumes={"/data": volume})
def read_data():
    volume.reload()  # Get latest changes
    with open("/data/output.txt") as f:
        return f.read()
```

### Key Operations

```python
# Commit changes (required for persistence)
volume.commit()

# Reload to see changes from other containers
volume.reload()

# Batch upload from local
with volume.batch_upload() as batch:
    batch.put_file("local.txt", "/remote/file.txt")
    batch.put_directory("./local_dir", "/remote/dir")
```

### CLI Commands

```bash
modal volume list                    # List volumes
modal volume create my-vol           # Create volume
modal volume ls my-vol              # List files
modal volume put my-vol ./local /remote  # Upload
modal volume get my-vol /remote ./local  # Download
modal volume rm my-vol /path         # Delete files
```

### Best Practices

- **Write-once, read-many**: Optimized for this pattern
- **Avoid concurrent writes**: Last write wins (no locking)
- **Use for model weights**: Cache downloaded models
- **Commit after writes**: Ensures persistence
- **Reload before reads**: To see changes from other containers

### Model Weights Pattern

```python
volume = modal.Volume.from_name("model-cache", create_if_missing=True)

def download_model():
    from huggingface_hub import snapshot_download
    snapshot_download("model-id", local_dir="/models/my-model")

image = (
    modal.Image.debian_slim()
    .pip_install("huggingface-hub")
    .run_function(download_model, volumes={"/models": volume})
)

@app.cls(volumes={"/models": volume})
class Model:
    @modal.enter()
    def load(self):
        self.model = load_model("/models/my-model")
```

## Secrets

Secure credential storage.

### Create Secrets

**Dashboard (Recommended):**
1. Go to Modal Dashboard → Secrets
2. Use templates for common services (AWS, HuggingFace, etc.)

**CLI:**
```bash
modal secret create my-secret KEY1=value1 KEY2=value2
modal secret create db-creds PGHOST=host PGUSER=user PGPASS="$PGPASS"
```

**Code:**
```python
modal.Secret.objects.create("my-secret", {"KEY": "value"})
```

### Use Secrets

```python
@app.function(secrets=[modal.Secret.from_name("my-secret")])
def use_secret():
    import os
    api_key = os.environ["API_KEY"]
```

### Multiple Secrets

```python
@app.function(secrets=[
    modal.Secret.from_name("openai-secret"),
    modal.Secret.from_name("db-secret"),
])
def multi_secret():
    pass
```

### From Environment/Dotenv

```python
# From local environment
@app.function(secrets=[modal.Secret.from_local_environ(["API_KEY", "DB_URL"])])
def from_env():
    pass

# From .env file
@app.function(secrets=[modal.Secret.from_dotenv()])
def from_dotenv():
    pass
```

### Inline Secrets (Development)

```python
@app.function(secrets=[modal.Secret.from_dict({"DEBUG": "1"})])
def debug_function():
    pass
```

## Cloud Bucket Mounts

Mount S3, GCS, or R2 buckets directly:

### S3

```python
bucket = modal.CloudBucketMount(
    "my-bucket",
    secret=modal.Secret.from_name("aws-secret"),
    read_only=True,
)

@app.function(volumes={"/data": bucket})
def read_s3():
    with open("/data/file.txt") as f:
        return f.read()
```

### Cloudflare R2 (Recommended for large datasets)

```python
bucket = modal.CloudBucketMount(
    "my-bucket",
    bucket_endpoint_url="https://xxx.r2.cloudflarestorage.com",
    secret=modal.Secret.from_name("r2-secret"),
)

@app.function(
    volumes={"/data": bucket},
    ephemeral_disk=1_000_000,  # 1TB temp disk
)
def process_large_dataset():
    pass
```

### GCS

```python
bucket = modal.CloudBucketMount(
    "my-gcs-bucket",
    secret=modal.Secret.from_name("gcp-secret"),
)
```

## Passing Local Data

### Function Arguments

```python
@app.function()
def process(data: bytes) -> dict:
    return {"size": len(data)}

# Call with local data
with open("file.bin", "rb") as f:
    result = process.remote(f.read())
```

### For Large Files

Use Volumes or Cloud Buckets:

```python
volume = modal.Volume.from_name("uploads", create_if_missing=True)

def upload_and_process(local_path: str):
    # Upload
    with volume.batch_upload() as batch:
        batch.put_file(local_path, "/input/data.bin")
    
    # Process remotely
    return process_file.remote("/input/data.bin")

@app.function(volumes={"/input": volume})
def process_file(path: str):
    with open(path, "rb") as f:
        return analyze(f.read())
```

### Image Files to Container

```python
image = (
    modal.Image.debian_slim()
    .add_local_dir("./data", remote_path="/app/data")
    .add_local_file("config.json", remote_path="/app/config.json")
)
```

## Storage Comparison

| Storage Type | Best For | Access Pattern |
|--------------|----------|----------------|
| Volumes | Model weights, datasets | Write-once, read-many |
| Secrets | Credentials, API keys | Read-only |
| Cloud Buckets | Existing cloud data | Direct mount |
| Dicts | Caching, distributed state | Random KV access |
| Queues | Task distribution | FIFO |
| Image files | Config, small assets | Read-only |
