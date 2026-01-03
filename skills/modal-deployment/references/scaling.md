# Scaling Reference

## Table of Contents
- [Autoscaling](#autoscaling)
- [Parallel Processing](#parallel-processing)
- [Input Concurrency](#input-concurrency)
- [Batch Processing](#batch-processing)
- [Dynamic Batching](#dynamic-batching)
- [Cold Start Optimization](#cold-start-optimization)
- [Limits](#limits)

## Autoscaling

Modal automatically scales containers based on demand.

### Scaling Parameters

```python
@app.function(
    min_containers=0,       # Minimum always-running containers (default: 0)
    max_containers=100,     # Maximum containers (default: system limit)
    buffer_containers=5,    # Extra warm containers for bursts
    scaledown_window=300,   # Seconds before idle container terminates
)
def autoscaled_function():
    pass
```

### Dynamic Autoscaler Updates

```python
# Update without redeploying
fn = modal.Function.from_name("my-app", "my_function")
fn.update_autoscaler(min_containers=5, max_containers=50)

# Schedule-based scaling
@app.function(schedule=modal.Cron("0 8 * * 1-5"))
def morning_warmup():
    fn = modal.Function.from_name("my-app", "inference")
    fn.update_autoscaler(min_containers=10)  # More during business hours

@app.function(schedule=modal.Cron("0 20 * * *"))
def evening_cooldown():
    fn = modal.Function.from_name("my-app", "inference")
    fn.update_autoscaler(min_containers=1)  # Less at night
```

## Parallel Processing

### .map() - Process Items in Parallel

```python
@app.function()
def process_item(item: str) -> dict:
    return {"item": item, "result": analyze(item)}

@app.local_entrypoint()
def main():
    items = ["a", "b", "c", "d", "e"]
    
    # Synchronous - blocks until all complete
    results = list(process_item.map(items))
    
    # Multiple arguments
    results = list(process_item.map(items_a, items_b))
```

### .spawn() - Fire and Forget

```python
# Submit job without waiting
call = process_item.spawn("data")

# Get result later
result = call.get()  # Blocks until complete
result = call.get(timeout=10)  # With timeout

# Check if complete without blocking
try:
    result = call.get(timeout=0)
except TimeoutError:
    print("Still processing...")
```

### .spawn_map() - Massive Batch Jobs

```python
# Submit up to 1 million inputs
@app.local_entrypoint()
def main():
    process_video.spawn_map(range(1_000_000))
```

Run with `--detach` to avoid waiting:
```bash
modal run --detach batch_job.py
```

### .for_each() - No Return Values

```python
# Process items, ignore returns (faster if you don't need results)
send_notification.for_each(user_ids)
```

### Async Patterns

```python
import asyncio

@app.function()
async def async_process(x: int) -> int:
    await asyncio.sleep(0.1)
    return x * 2

@app.local_entrypoint()
async def main():
    # Async iteration
    async for result in async_process.map.aio(range(100)):
        print(result)
    
    # Gather pattern
    calls = [async_process.spawn(i) for i in range(10)]
    results = await asyncio.gather(*[c.get.aio() for c in calls])
```

## Input Concurrency

Allow single containers to handle multiple concurrent inputs:

```python
@app.function()
@modal.concurrent(max_inputs=100, target_inputs=80)
async def fetch_url(url: str):
    """Each container handles up to 100 concurrent requests"""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()
```

Best for:
- I/O-bound work (API calls, database queries)
- Web endpoints
- GPU inference with continuous batching

### With Web Endpoints

```python
@app.function()
@modal.concurrent(max_inputs=100)
@modal.asgi_app()
def web_app():
    from fastapi import FastAPI
    app = FastAPI()
    # Routes...
    return app
```

## Batch Processing

### Job Queue Pattern

```python
# Submit jobs
@app.function(retries=2, timeout=3600)
def process_job(job_id: str):
    # Long-running processing
    return result

# Submit and track
@app.local_entrypoint()
def main():
    job_ids = ["job1", "job2", "job3"]
    calls = [process_job.spawn(jid) for jid in job_ids]
    
    # Track completion
    for call in calls:
        try:
            result = call.get()
            print(f"Completed: {result}")
        except Exception as e:
            print(f"Failed: {e}")
```

### With Detach for Long Jobs

```bash
# Don't wait for completion
modal run --detach batch_job.py
```

Results available via `FunctionCall.from_id()` for 7 days.

## Dynamic Batching

Accumulate requests into batches for GPU efficiency:

```python
@app.function(gpu="H100")
@modal.batched(max_batch_size=32, wait_ms=100)
async def batch_embed(texts: list[str]) -> list[list[float]]:
    """
    Receives texts as a batch.
    max_batch_size: Max items per batch
    wait_ms: Max wait time for batch to fill
    """
    return model.encode(texts)

# Call with single items - automatically batched
result = batch_embed.remote.aio("single text")
```

Configuration tips:
- `max_batch_size`: Set to largest batch your GPU can handle
- `wait_ms`: Target latency minus processing time

## Cold Start Optimization

### Use @modal.enter() for Initialization

```python
@app.cls(gpu="H100")
class Model:
    @modal.enter()
    def load(self):
        # Runs once per container, before accepting requests
        self.model = load_heavy_model()
```

### Memory Snapshots

```python
@app.function(enable_memory_snapshot=True)
def fast_start():
    import torch  # Snapshot includes import state
    pass
```

For GPU workloads:
```python
@app.cls(
    gpu="A10G",
    enable_memory_snapshot=True,
    experimental_options={"enable_gpu_snapshot": True}
)
class GPUModel:
    @modal.enter(snap=True)
    def load(self):
        self.model = load_to_gpu()  # GPU state snapshotted
```

### Concurrent File Loading

```python
import concurrent.futures

@modal.enter()
def load_models(self):
    def load_model(name):
        return AutoModel.from_pretrained(name)
    
    with concurrent.futures.ThreadPoolExecutor() as executor:
        futures = {executor.submit(load_model, n): n for n in model_names}
        self.models = {
            futures[f]: f.result() 
            for f in concurrent.futures.as_completed(futures)
        }
```

## Limits

| Limit | Value | Notes |
|-------|-------|-------|
| Max pending inputs | 2,000 | Per function (normal calls) |
| Max pending inputs (spawn) | 1,000,000 | For .spawn()/.spawn_map() |
| Max total inputs | 25,000 | Running + pending |
| Max concurrent .map() | 1,000 | Per invocation |
| Rate limit (new accounts) | 200/sec | Burst: 5x for 5 seconds |
| FunctionCall retention | 7 days | Results accessible via .get() |

Contact Modal to increase limits for production workloads.
