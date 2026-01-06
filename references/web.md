# Web Endpoints Reference

## Table of Contents
- [Quick Start](#quick-start)
- [FastAPI Endpoint](#fastapi-endpoint)
- [ASGI Apps](#asgi-apps)
- [WSGI Apps](#wsgi-apps)
- [Streaming](#streaming)
- [Authentication](#authentication)
- [Custom Domains](#custom-domains)
- [Timeouts](#timeouts)

## Quick Start

```python
import modal

app = modal.App("web-api")

@app.function()
@modal.fastapi_endpoint()
def hello(name: str = "world"):
    return {"message": f"Hello, {name}!"}
```

```bash
modal serve app.py    # Development (hot-reload)
modal deploy app.py   # Production
```

## FastAPI Endpoint

Simple single-endpoint functions:

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float

@app.function()
@modal.fastapi_endpoint(method="POST", docs=True)
def create_item(item: Item):
    return {"created": item.name, "price": item.price}
```

- `method`: HTTP method (GET, POST, etc.)
- `docs=True`: Enable Swagger UI at `/docs`
- CORS enabled automatically

## ASGI Apps

For complex applications with multiple routes:

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

@app.function()
@modal.concurrent(max_inputs=100)  # Handle concurrent requests
@modal.asgi_app()
def fastapi_app():
    web_app = FastAPI()
    
    @web_app.get("/")
    def root():
        return {"status": "ok"}
    
    @web_app.post("/predict")
    async def predict(request: Request):
        data = await request.json()
        return {"prediction": model.predict(data)}
    
    @web_app.get("/health")
    def health():
        return {"healthy": True}
    
    return web_app
```

### With Class Lifecycle

```python
@app.cls()
class WebService:
    @modal.enter()
    def load_model(self):
        self.model = load_model()
    
    @modal.asgi_app()
    def app(self):
        web_app = FastAPI()
        
        @web_app.post("/predict")
        def predict(data: dict):
            return self.model.predict(data)
        
        return web_app
```

### ASGI Lifespan

```python
@app.function()
@modal.asgi_app()
def app_with_lifespan():
    from contextlib import asynccontextmanager
    
    @asynccontextmanager
    async def lifespan(app):
        print("Starting up...")
        yield
        print("Shutting down...")
    
    web_app = FastAPI(lifespan=lifespan)
    return web_app
```

## WSGI Apps

For Flask and other WSGI frameworks:

```python
@app.function()
@modal.concurrent(max_inputs=100)
@modal.wsgi_app()
def flask_app():
    from flask import Flask, request
    
    web_app = Flask(__name__)
    
    @web_app.route("/")
    def index():
        return {"message": "Hello from Flask!"}
    
    @web_app.route("/echo", methods=["POST"])
    def echo():
        return request.json
    
    return web_app
```

## Web Server

For applications that bind to a port:

```python
@app.function()
@modal.web_server(port=8000)
def gradio_app():
    import gradio as gr
    
    demo = gr.Interface(fn=predict, inputs="text", outputs="text")
    demo.launch(server_name="0.0.0.0", server_port=8000)
```

## Streaming

### Server-Sent Events

```python
from fastapi.responses import StreamingResponse

@app.function()
@modal.fastapi_endpoint()
def stream():
    def generate():
        for i in range(10):
            yield f"data: {i}\n\n"
    return StreamingResponse(generate(), media_type="text/event-stream")
```

### Streaming with LLMs

```python
@app.cls(gpu="H100")
class LLM:
    @modal.enter()
    def load(self):
        self.model = load_model()
    
    @modal.fastapi_endpoint()
    def generate(self, prompt: str):
        from fastapi.responses import StreamingResponse
        
        def stream():
            for token in self.model.stream(prompt):
                yield f"data: {token}\n\n"
        
        return StreamingResponse(stream(), media_type="text/event-stream")
```

## Authentication

### Bearer Token

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

auth_scheme = HTTPBearer()

@app.function(secrets=[modal.Secret.from_name("api-auth")])
@modal.fastapi_endpoint()
def protected(token: HTTPAuthorizationCredentials = Depends(auth_scheme)):
    import os
    if token.credentials != os.environ["AUTH_TOKEN"]:
        raise HTTPException(status_code=401, detail="Invalid token")
    return {"message": "Authenticated!"}
```

### Proxy Auth Tokens

Modal-managed authentication for endpoints:

```python
@app.function()
@modal.fastapi_endpoint(requires_proxy_auth=True)
def protected():
    return {"secret": "data"}
```

Clients authenticate with headers:
```bash
curl -H "Modal-Key: $TOKEN_ID" -H "Modal-Secret: $TOKEN_SECRET" $URL
```

Create tokens in Modal Dashboard → Settings → Proxy Auth Tokens.

## Custom Domains

```python
@app.function()
@modal.fastapi_endpoint(custom_domains=["api.example.com"])
def api():
    return {"status": "ok"}
```

Setup:
1. Add domain in Modal Dashboard
2. Configure DNS CNAME to Modal
3. TLS certificates auto-provisioned

Multiple domains:
```python
@modal.fastapi_endpoint(custom_domains=["api.example.com", "api.example.net"])
```

## URL Structure

Default URL format:
```
https://{workspace}--{app-name}-{function-name}.modal.run
```

Custom label:
```python
@modal.fastapi_endpoint(label="v2-api")
def endpoint():
    pass
# URL: https://{workspace}--v2-api.modal.run
```

Development URLs have `-dev` suffix:
```
https://{workspace}--{function-name}-dev.modal.run
```

## Timeouts

Web requests timeout after 150 seconds, but:
- Modal returns 303 redirect to result URL
- Browsers follow up to 20 redirects (~50 min effective timeout)

### Long-running Jobs Pattern

```python
from fastapi import FastAPI
import modal

web_app = FastAPI()

@app.function()
def slow_job(data):
    # Long processing...
    return result

@app.function()
@modal.asgi_app()
def api():
    @web_app.post("/submit")
    def submit(data: dict):
        call = slow_job.spawn(data)
        return {"job_id": call.object_id}
    
    @web_app.get("/result/{job_id}")
    def get_result(job_id: str):
        call = modal.FunctionCall.from_id(job_id)
        try:
            return {"result": call.get(timeout=0)}
        except TimeoutError:
            return {"status": "pending"}, 202
    
    return web_app
```

## Static Files

```python
from fastapi.staticfiles import StaticFiles

image = modal.Image.debian_slim().add_local_dir("./static", remote_path="/static")

@app.function(image=image)
@modal.asgi_app()
def app():
    web_app = FastAPI()
    web_app.mount("/static", StaticFiles(directory="/static"), name="static")
    return web_app
```

## Rate Limits

- New accounts: 200 requests/second (5x burst)
- 429 status returned when exceeded
- Contact Modal to increase limits
