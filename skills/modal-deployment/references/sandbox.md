# Sandbox Reference

Isolated containers for running untrusted code safely.

## Quick Start

```python
import modal

app = modal.App.lookup("my-app", create_if_missing=True)

sb = modal.Sandbox.create(app=app, timeout=300)
process = sb.exec("python", "-c", "print('hello')")
print(process.stdout.read())
sb.terminate()
```

Run directly with `python script.py` (no `modal run`).

## Creating Sandboxes

```python
# With custom image and resources
image = modal.Image.debian_slim().pip_install("numpy", "pandas")
sb = modal.Sandbox.create(
    app=app,
    image=image,
    timeout=3600,
    cpu=4,
    memory=8192,
    gpu="A100",
)

# With volumes and secrets
sb = modal.Sandbox.create(
    app=app,
    volumes={"/data": modal.Volume.from_name("my-vol")},
    secrets=[modal.Secret.from_name("my-secret")],
)
```

## Executing Commands

```python
# Run command and read output
process = sb.exec("python", "-c", code, timeout=10)
stdout = process.stdout.read()
stderr = process.stderr.read()

# Stream output
for line in process.stdout:
    print(line, end="")
process.wait()

# With environment and workdir
process = sb.exec("python", "script.py", env={"MY_VAR": "val"}, workdir="/app")
```

## File Access

```python
# Write file
with sb.open("/app/data.txt", "w") as f:
    f.write("content")

# Read file
content = sb.open("/app/data.txt", "r").read()

# Directory operations
sb.mkdir("/app/subdir")
files = sb.ls("/app")
sb.rm("/app/data.txt")
```

## Networking

```python
# Block all network
sb = modal.Sandbox.create(app=app, block_network=True)

# Allow specific IPs only
sb = modal.Sandbox.create(app=app, cidr_allowlist=["10.0.0.0/8"])

# Expose port with tunnel
sb = modal.Sandbox.create(
    "python", "-m", "http.server", "8080",
    encrypted_ports=[8080],
    app=app,
)
tunnel = sb.tunnels()[8080]
print(tunnel.url)
```

## Snapshots

```python
# Create sandbox and install dependencies
sb = modal.Sandbox.create(app=app)
sb.exec("pip", "install", "pandas", "numpy").wait()

# Snapshot filesystem -> Image
snapshot = sb.snapshot_filesystem()
sb.terminate()

# Reuse snapshot for new sandboxes
sb2 = modal.Sandbox.create(app=app, image=snapshot)
```

## Safe Code Execution Pattern

```python
def run_untrusted(code: str) -> dict:
    app = modal.App.lookup("runner", create_if_missing=True)
    sb = modal.Sandbox.create(
        app=app,
        timeout=30,
        block_network=True,
        memory=512,
    )
    try:
        with sb.open("/tmp/code.py", "w") as f:
            f.write(code)
        process = sb.exec("python", "/tmp/code.py", timeout=10)
        return {"stdout": process.stdout.read(), "stderr": process.stderr.read()}
    finally:
        sb.terminate()
```

## Best Practices

1. **Always terminate** - Use try/finally for cleanup
2. **Set timeouts** - Prevent runaway sandboxes
3. **Block network** - For untrusted code
4. **Use snapshots** - Reuse setup across sandboxes
5. **Limit resources** - Constrain CPU/memory
