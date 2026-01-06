# Dict Reference

Distributed key-value store for sharing state across containers.

## Quick Start

```python
import modal

cache = modal.Dict.from_name("my-cache", create_if_missing=True)

cache["key"] = {"data": [1, 2, 3]}
value = cache["key"]
```

## Basic Operations

```python
cache = modal.Dict.from_name("cache", create_if_missing=True)

# Set and get
cache["user:123"] = {"name": "Alice", "score": 100}
user = cache["user:123"]
user = cache.get("user:999", default=None)

# Check existence
if "user:123" in cache:
    print("exists")

# Delete
del cache["user:123"]
value = cache.pop("key", default=None)

# Update multiple
cache.update({"key1": "val1", "key2": "val2"})

# Clear all
cache.clear()
```

## Atomic Operations (Distributed Locking)

```python
# Put only if key doesn't exist (atomic)
was_added = cache.put("lock:resource", True, skip_if_exists=True)

if was_added:
    try:
        # Do exclusive work
        pass
    finally:
        cache.pop("lock:resource", default=None)
```

## Async Operations

```python
cache = modal.Dict.from_name("cache", create_if_missing=True)

async def async_ops():
    await cache.put.aio("key", "value")
    value = await cache.get.aio("key")
    exists = await cache.contains.aio("key")
```

## Common Patterns

### Caching

```python
@app.function()
def cached_compute(key: str):
    cached = cache.get(key, default=None)
    if cached is not None:
        return cached
    result = expensive_computation(key)
    cache[key] = result
    return result
```

### Session State

```python
sessions = modal.Dict.from_name("sessions", create_if_missing=True)

@app.function()
def handle_request(session_id: str, data: dict):
    session = sessions.get(session_id, default={})
    session["last_request"] = data
    sessions[session_id] = session
    return session
```

## Best Practices

1. **Use descriptive key prefixes** - `user:123`, `cache:embeddings:v1`
2. **Handle missing keys** - Use `.get(key, default)` instead of `[key]`
3. **Use skip_if_exists for locks** - Atomic operation prevents races
4. **Use async in web endpoints** - `.aio` methods avoid blocking
5. **Keep values small** - Smaller is faster

## Limitations

- Not a database (no queries, indexes, transactions)
- Eventually consistent
- 100 KB max per value
