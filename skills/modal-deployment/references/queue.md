# Queue Reference

Distributed FIFO queues for passing data between containers.

## Quick Start

```python
import modal

queue = modal.Queue.from_name("my-queue", create_if_missing=True)

# Producer
queue.put({"task": "process", "id": 123})

# Consumer (blocks until message available)
message = queue.get()
```

## Basic Operations

```python
queue = modal.Queue.from_name("queue", create_if_missing=True)

# Put single or multiple items
queue.put("message")
queue.put_many(["msg1", "msg2", "msg3"])

# Get with timeout (returns None if timeout)
message = queue.get(timeout=30)

# Non-blocking get
message = queue.get(timeout=0)

# Clear queue
queue.clear()

# Get length
count = queue.len()
```

## Partitions

Independent FIFO queues within one Queue:

```python
# Put to named partitions
queue.put("high priority", partition="high")
queue.put("low priority", partition="low")

# Get from specific partition
high = queue.get(partition="high")
low = queue.get(partition="low")

# Custom TTL per partition (default 24h)
queue.put("temp", partition="ephemeral", partition_ttl=60)
```

## Async Operations

```python
async def async_ops():
    await queue.put.aio("message")
    message = await queue.get.aio(timeout=30)
```

## Common Patterns

### Task Queue

```python
@app.function()
def producer(items: list):
    for item in items:
        task_queue.put(item)
    # Signal completion
    task_queue.put(None)

@app.function()
def worker():
    while True:
        task = task_queue.get(timeout=60)
        if task is None:
            break
        process(task)
```

### Fan-Out / Fan-In

```python
@app.function()
def coordinator(items: list):
    with modal.Queue.ephemeral() as results:
        # Fan out
        for item in items:
            process_item.spawn(item, results)

        # Fan in
        return [results.get(timeout=300) for _ in items]

@app.function()
def process_item(item, result_queue: modal.Queue):
    result_queue.put(do_work(item))
```

## Best Practices

1. **Use timeouts** - Always specify timeout in `get()` to prevent blocking forever
2. **Signal completion** - Use sentinel values (None) to signal consumers to stop
3. **Use put_many** - More efficient than multiple puts
4. **Use partitions** - Isolate different workloads
5. **Use ephemeral queues** - For one-time coordination

## Limitations

- Not durable (data may be lost on failures)
- 24-hour TTL per partition
- 5,000 items per partition
- 1 MiB per item
- No message acknowledgment
