# Networking Reference

## Table of Contents
- [Tunnels](#tunnels)
- [Proxies](#proxies)
- [Cluster Networking (i6pn)](#cluster-networking-i6pn)
- [Common Patterns](#common-patterns)

## Tunnels

Expose live TCP ports on containers to the public internet. Ideal for:
- Jupyter notebooks
- VS Code servers
- Interactive debugging
- Live web apps

### Basic Tunnel

```python
import modal

app = modal.App()

@app.function()
def start_server():
    with modal.forward(8000) as tunnel:
        print(f"URL: {tunnel.url}")
        print(f"TLS socket: {tunnel.tls_socket}")
        # Start your server on port 8000
        run_server(port=8000)
```

Within milliseconds, you get a public URL like: `https://abc123.r5.modal.host`

### Jupyter Notebook Server

```python
import os
import secrets
import subprocess
import modal

app = modal.App(image=modal.Image.debian_slim().pip_install("jupyterlab"))

@app.function(timeout=3600)
def run_jupyter():
    token = secrets.token_urlsafe(13)
    
    with modal.forward(8888) as tunnel:
        url = f"{tunnel.url}/?token={token}"
        print(f"Jupyter available at: {url}")
        
        subprocess.run([
            "jupyter", "lab",
            "--no-browser", "--allow-root",
            "--ip=0.0.0.0", "--port=8888",
            f"--LabApp.token={token}",
        ], env={**os.environ, "SHELL": "/bin/bash"})
```

### SSH into Container

```python
import subprocess
import time
import modal

app = modal.App()

image = (
    modal.Image.debian_slim()
    .apt_install("openssh-server")
    .run_commands("mkdir /run/sshd")
    .add_local_file("~/.ssh/id_rsa.pub", "/root/.ssh/authorized_keys", copy=True)
)

@app.function(image=image, timeout=3600)
def ssh_server():
    subprocess.Popen(["/usr/sbin/sshd", "-D", "-e"])
    
    with modal.forward(port=22, unencrypted=True) as tunnel:
        hostname, port = tunnel.tcp_socket
        print(f"SSH: ssh -p {port} root@{hostname}")
        time.sleep(3600)  # Keep alive
```

### TCP Echo Server

```python
import socket
import threading
import modal

def run_echo_server(port: int):
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.bind(("0.0.0.0", port))
    sock.listen(1)
    
    while True:
        conn, addr = sock.accept()
        def handle(c):
            with c:
                while data := c.recv(1024):
                    c.sendall(data)
        threading.Thread(target=handle, args=(conn,)).start()

@app.function()
def tcp_tunnel():
    with modal.forward(8000, unencrypted=True) as tunnel:
        print(f"TCP: {tunnel.tcp_socket}")  # (host, port)
        run_echo_server(8000)
```

### Tunnel Options

```python
# Encrypted TLS (default)
with modal.forward(8000) as tunnel:
    pass

# Unencrypted TCP (for SSH, raw TCP protocols)
with modal.forward(8000, unencrypted=True) as tunnel:
    pass

# HTTP/2 support
with modal.forward(8000, h2_enabled=True) as tunnel:
    pass
```

### On-Demand Jupyter Hub

```python
@app.function(timeout=900)
def run_jupyter(result_queue: modal.Queue):
    token = secrets.token_urlsafe(13)
    with modal.forward(8888) as tunnel:
        url = f"{tunnel.url}/?token={token}"
        result_queue.put(url)  # Send URL back
        run_jupyter_server(token)

@app.function()
@modal.fastapi_endpoint(method="POST")
def jupyter_hub():
    from fastapi.responses import RedirectResponse
    
    # Validate authentication...
    with modal.Queue.ephemeral() as q:
        run_jupyter.spawn(q)
        url = q.get(timeout=60)
        return RedirectResponse(url, status_code=303)
```

### Tunnels with Sandboxes

```python
import requests
import time

sb = modal.Sandbox.create(
    "python", "-m", "http.server", "8080",
    encrypted_ports=[8080],
    app=app,
)

time.sleep(1)  # Wait for server start
tunnel = sb.tunnels()[8080]
print(f"Server at: {tunnel.url}")

response = requests.get(tunnel.url)
print(response.text)

sb.terminate()
```

## Proxies

Static outbound IP addresses for connecting to resources with IP whitelisting:

### Setup

1. Create proxy in Dashboard (Settings → Proxies)
2. Add to function:

```python
@app.function(proxy=modal.Proxy.from_name("my-proxy"))
def call_whitelisted_api():
    import subprocess
    # All traffic goes through proxy with static IP
    subprocess.run(["curl", "-s", "https://api.example.com"])
```

### With Sandboxes

```python
sb = modal.Sandbox.create(
    app=app,
    image=modal.Image.debian_slim().apt_install("curl"),
    proxy=modal.Proxy.from_name("my-proxy"),
)

p = sb.exec("curl", "-s", "https://ifconfig.me")
print(p.stdout.read())  # Always same IP
```

### Multiple IPs

Add up to 5 IPs per proxy for higher throughput (linear improvement).

## Cluster Networking (i6pn)

Private container-to-container networking with:
- Low latency
- High bandwidth (≥50 Gbps)
- IPv6 private addresses

### Enable i6pn

```python
@app.function(i6pn=True)
def get_private_address():
    import socket
    # Get own IPv6 address
    addr = socket.getaddrinfo("i6pn.modal.local", None, socket.AF_INET6)[0][4][0]
    print(f"My address: {addr}")  # fdaa:5137:3ebf:a70:...
```

### Multi-Node Clusters

```python
@app.function(i6pn=True)
@modal.experimental.clustered(n_containers=4)
def distributed_training():
    import socket
    
    # Each container gets unique private IPv6
    my_addr = socket.getaddrinfo("i6pn.modal.local", None, socket.AF_INET6)[0][4][0]
    
    # Containers can communicate directly
    # Use for distributed ML training, parallel processing, etc.
```

### Region Constraints

i6pn is region-scoped. Clustered containers run in the same region automatically.

### Public Access via Tunnel

i6pn addresses are private. Use Tunnels to expose a gateway:

```python
@app.function(i6pn=True)
def cluster_gateway():
    with modal.forward(8000) as tunnel:
        print(f"Public access: {tunnel.url}")
        # Route traffic to cluster
```

## Common Patterns

### Live Debugging with VS Code

```python
app = modal.App(image=modal.Image.debian_slim().pip_install("code-server"))

@app.function(timeout=3600)
def vscode_server():
    import subprocess
    
    with modal.forward(8080) as tunnel:
        print(f"VS Code at: {tunnel.url}")
        subprocess.run(["code-server", "--bind-addr=0.0.0.0:8080", "--auth=none"])
```

### Database Connection through Proxy

```python
@app.function(
    proxy=modal.Proxy.from_name("db-proxy"),
    secrets=[modal.Secret.from_name("db-creds")],
)
def query_database():
    import os
    import psycopg2
    
    # Connection goes through static IP (whitelisted by DB)
    conn = psycopg2.connect(
        host=os.environ["DB_HOST"],
        user=os.environ["DB_USER"],
        password=os.environ["DB_PASSWORD"],
    )
    return conn.execute("SELECT 1").fetchone()
```

### Distributed Training with i6pn

```python
@app.function(gpu="H100:8", i6pn=True)
@modal.experimental.clustered(n_containers=4)
def distributed_llm_training():
    import torch.distributed as dist
    
    # 4 nodes × 8 GPUs = 32 GPU training
    # i6pn provides fast interconnect
    dist.init_process_group(backend="nccl")
    train_model()
```

### Tunnel Properties

```python
with modal.forward(8000) as tunnel:
    tunnel.url          # https://abc123.r5.modal.host
    tunnel.tls_socket   # ('abc123.r5.modal.host', 443)
    tunnel.tcp_socket   # ('abc123.r5.modal.host', 12345) if unencrypted
```

## Best Practices

1. **Use encrypted tunnels by default**: Only use `unencrypted=True` for protocols that require it (SSH, raw TCP)

2. **Set timeouts for tunnel functions**: Prevent runaway costs

3. **Use Proxies for external services**: When you need static IPs for whitelisting

4. **Use i6pn for high-bandwidth internal communication**: Much faster than going through public internet

5. **Secure tunnel URLs**: While cryptographically random, anyone with the URL can access. Add your own auth layer.

6. **Combine with Queues for coordination**: Pass tunnel URLs back to callers via Queues

## Limitations

- **Tunnel URLs are public**: No built-in auth (add your own)
- **No L7 detection**: Raw TCP/TLS only, no HTTP header injection
- **i6pn is region-scoped**: Cross-region communication not supported
- **Proxies add latency**: Encrypted WireGuard tunnels have overhead
