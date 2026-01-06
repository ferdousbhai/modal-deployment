# Functions & Classes Reference

## Table of Contents
- [Basic Functions](#basic-functions)
- [Function Parameters](#function-parameters)
- [Classes with Lifecycle](#classes-with-lifecycle)
- [Invocation Methods](#invocation-methods)
- [Entrypoints](#entrypoints)
- [Deploying Functions](#deploying-functions)
- [Scheduling (Cron Jobs)](#scheduling-cron-jobs)

## Basic Functions

```python
import modal

app = modal.App("my-app")

@app.function()
def simple_function(x: int) -> int:
    return x * 2
```

## Function Parameters

```python
@app.function(
    # Compute resources
    cpu=2.0,              # CPU cores (float, default 0.125)
    memory=4096,          # Memory in MiB
    gpu="H100",           # GPU type
    ephemeral_disk=10000, # Disk in MiB
    
    # Container image
    image=my_image,
    
    # Execution settings
    timeout=600,          # Max runtime in seconds (default 300)
    retries=3,            # Auto-retry on failure
    
    # Scaling
    min_containers=1,     # Minimum warm containers
    max_containers=10,    # Maximum containers
    buffer_containers=2,  # Extra buffer for bursts
    scaledown_window=60,  # Seconds before scale-down
    
    # Attachments
    secrets=[modal.Secret.from_name("api-key")],
    volumes={"/data": volume},
    
    # Scheduling
    schedule=modal.Period(hours=1),
)
def configured_function():
    pass
```

## Classes with Lifecycle

Use classes when you need:
- Expensive initialization (model loading)
- Shared state across calls
- Multiple related methods

### Basic Class

```python
@app.cls(gpu="L40S", image=ml_image)
class TextGenerator:
    @modal.enter()
    def setup(self):
        """Called once when container starts"""
        from transformers import pipeline
        self.pipe = pipeline("text-generation", device="cuda")
    
    @modal.method()
    def generate(self, prompt: str) -> str:
        return self.pipe(prompt)[0]["generated_text"]
    
    @modal.exit()
    def cleanup(self):
        """Called when container shuts down"""
        del self.pipe

# Usage
gen = TextGenerator()
result = gen.generate.remote("Hello")
```

### Async Enter

```python
@app.cls()
class AsyncModel:
    @modal.enter()
    async def setup(self):
        self.client = await create_async_client()
```

### Multiple Enter Methods

```python
@app.cls()
class MultiSetup:
    @modal.enter()
    def load_model(self):
        self.model = load_model()
    
    @modal.enter()
    def load_tokenizer(self):
        self.tokenizer = load_tokenizer()
```

### Memory Snapshots with Classes

```python
@app.cls(
    gpu="A10G",
    enable_memory_snapshot=True,
    experimental_options={"enable_gpu_snapshot": True}
)
class SnapshotModel:
    @modal.enter(snap=True)  # Include in snapshot
    def load(self):
        self.model = load_model_to_gpu()
    
    @modal.method()
    def infer(self, x):
        return self.model(x)
```

## Invocation Methods

### Remote Call (Synchronous)
```python
result = my_function.remote(arg1, arg2)
```

### Async Remote Call
```python
result = await my_function.remote.aio(arg1, arg2)
```

### Spawn (Fire and Forget)
```python
call = my_function.spawn(arg1, arg2)
# Later...
result = call.get()  # Wait for result
result = call.get(timeout=10)  # With timeout
```

### Map (Parallel Processing)
```python
# Synchronous
results = list(my_function.map(items))

# With multiple arguments
results = list(my_function.map(items_a, items_b))

# Async
async for result in my_function.map.aio(items):
    print(result)
```

### Spawn Map (Batch Jobs)
```python
# Submit up to 1M inputs for background processing
my_function.spawn_map(range(100_000))
```

### For Each (No Return Values)
```python
my_function.for_each(items)  # Runs all, ignores returns
```

## Entrypoints

### Local Entrypoint
Code runs locally, can call remote functions:

```python
@app.local_entrypoint()
def main(count: int = 10):
    results = list(process.map(range(count)))
    print(f"Processed {len(results)} items")
```

Run: `modal run app.py --count 100`

### Remote Entrypoint
Entire entrypoint runs remotely:

```python
@app.function()
def main():
    # This runs in the cloud
    return heavy_computation()
```

Run: `modal run app.py::main`

### Custom Argument Parsing

```python
@app.function()
def train(*arglist):
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument("--epochs", type=int, default=10)
    args = parser.parse_args(arglist)
```

## Deploying Functions

### Deploy
```bash
modal deploy app.py
```

Creates persistent deployment. Functions run on schedule or via invocation.

### Invoke Deployed Functions

**From Python:**
```python
fn = modal.Function.from_name("my-app", "my_function")
result = fn.remote(arg)
```

**From Other Languages (HTTP):**
```python
# First, add web endpoint
@app.function()
@modal.fastapi_endpoint()
def api_endpoint(x: int):
    return {"result": x * 2}
```

Then call via HTTPS.

### Stopping Apps
```bash
modal app stop my-app
```

Or via web UI.

## Function Lookups

Reference deployed functions:

```python
# Lookup by name
fn = modal.Function.from_name("app-name", "function-name")
result = fn.remote(args)

# Lookup class
cls = modal.Cls.from_name("app-name", "ClassName")
obj = cls()
result = obj.method.remote(args)
```

## Scheduling (Cron Jobs)

```python
# Interval-based (resets on redeploy)
@app.function(schedule=modal.Period(hours=1))
def hourly_task():
    pass

# Cron-based (fixed times, survives redeploy)
@app.function(schedule=modal.Cron("0 8 * * 1"))  # 8 AM UTC Mondays
def weekly_report():
    pass

# With timezone
@app.function(schedule=modal.Cron("0 6 * * *", timezone="America/New_York"))
def morning_report():
    pass
```

Activate with `modal deploy app.py`.

## Best Practices

1. **Use classes for ML models** - Load once in `@modal.enter()`
2. **Set appropriate timeouts** - Default is 5 minutes
3. **Use retries for flaky operations** - Network calls, API requests
4. **Spawn for background jobs** - Don't block on long tasks
5. **Map for parallelism** - Process items concurrently
6. **Use Cron for fixed schedules** - Survives redeployments
7. **Use Period for intervals** - Simple recurring tasks
