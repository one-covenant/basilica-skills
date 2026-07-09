# Serverless Deployments

Use this reference when a user wants to deploy an HTTP service, web app,
container, GPU app, stateful demo, WebSocket app, public-metadata deployment, or
many short-lived deployments on Basilica.

## Contents

- Surface Selection
- CLI Patterns
- SDK Basic HTTP Service
- SDK FastAPI File Deployment
- SDK Container Deployment
- SDK Decorator Deployment
- Storage And Volumes
- GPU App Deployment
- WebSockets
- Public Metadata
- Custom Commands
- Progress And Async Orchestration
- Troubleshooting

## Surface Selection

Use the highest-level surface that gives enough control:

1. Use `basilica deploy ...` for interactive CLI workflows and quick demos.
2. Use `client.deploy(...)` for scripts, CI, notebooks, and source/file/container
   deployments where the agent should wait for readiness.
3. Use `@basilica.deployment` when the app is naturally expressed as a Python
   function and should be deployable by calling that function.
4. Use `client.create_deployment(...)` for lower-level deployment features such
   as custom commands, WebSockets, public metadata, custom health checks,
   custom images, or when the script should create first and wait separately.
5. Use `client.deploy_async(...)` and async cleanup when launching many
   deployments concurrently.

Always make cost-bearing behavior explicit. Prefer `--ttl` or `ttl_seconds=`
for experiments, and show cleanup with `basilica deploy delete <name>` or
`deployment.delete()`.

## CLI Patterns

Deploy a Python file with dependencies:

```bash
basilica deploy my_api.py \
  --name my-api \
  --port 8000 \
  --pip fastapi uvicorn \
  --ttl 600
```

Deploy a non-root container image:

```bash
basilica deploy nginxinc/nginx-unprivileged:alpine \
  --name nginx-demo \
  --port 8080 \
  --cpu 250m \
  --memory 256Mi \
  --ttl 300
```

Deploy with GPU resources:

```bash
basilica deploy inference.py \
  --name gpu-model \
  --gpu 1 \
  --gpu-model H100 \
  --gpu-memory-gb 80 \
  --memory 32Gi \
  --pip torch \
  --ttl 3600
```

Deploy with persistent storage mounted at `/data`:

```bash
basilica deploy hello.py \
  --name stateful-app \
  --storage \
  --storage-path /data \
  --ttl 3600
```

Deploy with custom health checks:

```bash
basilica deploy my_api.py \
  --name health-api \
  --port 8000 \
  --pip fastapi uvicorn \
  --health-path /health \
  --health-initial-delay 10 \
  --health-period 30 \
  --ttl 600
```

Deploy with WebSocket support:

```bash
basilica deploy ws_app.py \
  --name ws-app \
  --port 8000 \
  --websocket \
  --ws-idle-timeout 3600 \
  --ttl 3600
```

Enroll deployment metadata for public validator verification:

```bash
basilica deploy hashicorp/http-echo:latest \
  --name metadata-demo \
  --port 5678 \
  --public-metadata \
  --ttl 600

basilica deploy enroll-metadata metadata-demo
basilica deploy metadata metadata-demo --json
```

Manage deployments:

```bash
basilica deploy ls --json
basilica deploy status my-api --show-phases
basilica deploy logs my-api --tail 100
basilica deploy logs my-api --follow
basilica deploy scale my-api --replicas 3
basilica deploy restart my-api
basilica deploy delete my-api --yes
```

## SDK Basic HTTP Service

Use inline source for small demos and prototypes:

```python
from basilica import BasilicaClient

client = BasilicaClient()

deployment = client.deploy(
    name="hello",
    source="""
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"Hello from Basilica!")

HTTPServer(("", 8000), Handler).serve_forever()
""",
    port=8000,
    ttl_seconds=600,
)

print(deployment.url)
deployment.delete()
```

## SDK FastAPI File Deployment

Use a source file for single-file apps. The SDK reads and packages the file.

```python
from basilica import BasilicaClient

client = BasilicaClient()

deployment = client.deploy(
    name="file-api",
    source="app_file.py",
    port=8000,
    pip_packages=["fastapi", "uvicorn"],
    ttl_seconds=600,
    timeout=180,
)

print(f"docs:   {deployment.url}/docs")
print(f"health: {deployment.url}/health")
deployment.delete()
```

The app file should bind to `0.0.0.0` on the deployed port:

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "healthy"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## SDK Container Deployment

Use `image=` when the app is already packaged. Basilica runs containers as a
non-root user, so choose images that work without root privileges.

```python
from basilica import BasilicaClient

client = BasilicaClient()

deployment = client.deploy(
    name="nginx-demo",
    image="nginxinc/nginx-unprivileged:alpine",
    port=8080,
    replicas=1,
    env={"NGINX_HOST": "localhost"},
    cpu="250m",
    memory="256Mi",
    ttl_seconds=600,
    timeout=120,
)

print(deployment.url)
deployment.delete()
```

For multi-file projects, build and push a custom image first:

```dockerfile
FROM python:3.11-slim

RUN useradd -m -u 1000 appuser
WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:appuser app/ ./app/
USER appuser

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
docker build -t ghcr.io/yourusername/my-api:latest .
docker push ghcr.io/yourusername/my-api:latest
```

```python
deployment = client.deploy(
    name="custom-api",
    image="ghcr.io/yourusername/my-api:latest",
    port=8000,
    ttl_seconds=3600,
    timeout=180,
)
```

## SDK Decorator Deployment

Use `@basilica.deployment` for Python functions that start their own HTTP
server.

```python
import basilica

@basilica.deployment(
    name="decorator-api",
    port=8000,
    pip_packages=["fastapi", "uvicorn"],
    ttl_seconds=600,
)
def serve():
    from fastapi import FastAPI
    import uvicorn

    app = FastAPI()

    @app.get("/")
    def root():
        return {"message": "Hello from decorator FastAPI!"}

    uvicorn.run(app, host="0.0.0.0", port=8000)

deployment = serve()
print(deployment.url)
deployment.delete()
```

## Storage And Volumes

Use `storage=True` for high-level deploys that need persistent data at `/data`.
Use `Volume.from_name(..., create_if_missing=True)` with decorator deployments.

```python
from basilica import BasilicaClient

client = BasilicaClient()

deployment = client.deploy(
    name="counter",
    source="""
from http.server import HTTPServer, BaseHTTPRequestHandler
from pathlib import Path

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        f = Path("/data/count")
        n = int(f.read_text()) + 1 if f.exists() else 1
        f.write_text(str(n))
        self.send_response(200)
        self.end_headers()
        self.wfile.write(f"Visit #{n}".encode())

HTTPServer(("", 8000), Handler).serve_forever()
""",
    port=8000,
    storage=True,
    ttl_seconds=600,
)
```

```python
import basilica

cache = basilica.Volume.from_name("counter-cache", create_if_missing=True)

@basilica.deployment(
    name="decorator-counter",
    port=8000,
    volumes={"/data": cache},
    ttl_seconds=600,
)
def serve():
    ...
```

## GPU App Deployment

Use the CUDA-capable image plus explicit GPU and memory requirements.

```python
from basilica import BasilicaClient

client = BasilicaClient()

deployment = client.deploy(
    name="gpu-test",
    source="""
import json
import torch
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        info = {
            "cuda_available": torch.cuda.is_available(),
            "device_count": torch.cuda.device_count(),
            "device_name": torch.cuda.get_device_name(0) if torch.cuda.is_available() else None,
        }
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        self.wfile.write(json.dumps(info).encode())

HTTPServer(("", 8000), Handler).serve_forever()
""",
    image="pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime",
    port=8000,
    gpu_count=1,
    min_gpu_memory_gb=16,
    memory="8Gi",
    ttl_seconds=600,
    timeout=300,
)

print(deployment.url)
deployment.delete()
```

Use placement preferences when needed:

```python
deployment = client.deploy(
    name="flavour-hello",
    source="app.py",
    port=8000,
    gpu_count=1,
    gpu_models=["H100"],
    interconnect="SXM",
    ttl_seconds=600,
)
```

## WebSockets

Use `WebSocketConfig` when the gateway must allow long-lived bidirectional
connections.

```python
from basilica import BasilicaClient, WebSocketConfig

client = BasilicaClient()

deployment = client.create_deployment(
    instance_name="ws-demo",
    image="hashicorp/http-echo:latest",
    replicas=1,
    port=5678,
    websocket=WebSocketConfig(enabled=True, idle_timeout_seconds=3600),
    ttl_seconds=600,
)

print(deployment.url)
client.delete_deployment(deployment.instance_name)
```

## Public Metadata

Use public metadata only when non-sensitive deployment metadata should be
publicly queryable for validator verification.

```python
from basilica import BasilicaClient

client = BasilicaClient()

deployment = client.create_deployment(
    instance_name="metadata-demo",
    image="hashicorp/http-echo:latest",
    replicas=1,
    port=5678,
    public_metadata=True,
    ttl_seconds=600,
)

status = client.get_enrollment_status(deployment.instance_name)
metadata = client.get_public_deployment_metadata(deployment.instance_name)

print(status.public_metadata)
print(metadata.state)

client.enroll_metadata(deployment.instance_name, enabled=False)
client.enroll_metadata(deployment.instance_name, enabled=True)
client.delete_deployment(deployment.instance_name)
```

## Custom Commands

Use `create_deployment()` when the app needs a specific container command.
This is useful for frameworks that need a custom runner.

```python
import base64
from pathlib import Path
from basilica import BasilicaClient

client = BasilicaClient()

app_source = Path("streamlit_app.py").read_text()
app_b64 = base64.b64encode(app_source.encode()).decode()

script = (
    "pip install -q streamlit && "
    f'echo "{app_b64}" | base64 -d > /tmp/app.py && '
    "python3 -m streamlit run /tmp/app.py "
    "--server.port=8501 --server.address=0.0.0.0 --server.headless=true"
)

response = client.create_deployment(
    instance_name="streamlit-demo",
    image="python:3.11-slim",
    port=8501,
    command=["bash", "-c", script],
    cpu="500m",
    memory="512Mi",
    ttl_seconds=3600,
)

deployment = client.get(response.instance_name)
deployment.wait_until_ready(timeout=300)
print(deployment.url)
deployment.delete()
```

## Progress And Async Orchestration

Use progress callbacks when building a UI, logging deployment stages, or
debugging startup phases.

```python
from basilica import BasilicaClient, DeploymentStatus

def on_progress(status: DeploymentStatus) -> None:
    phase = status.phase or "unknown"
    replicas = f"{status.replicas_ready}/{status.replicas_desired}"
    print(f"{phase} replicas={replicas}")

client = BasilicaClient()

response = client.create_deployment(
    instance_name="progress-demo",
    image="python:3.11-slim",
    command=[
        "python",
        "-c",
        "from http.server import HTTPServer, BaseHTTPRequestHandler; "
        "HTTPServer(('', 8000), type('H', (BaseHTTPRequestHandler,), "
        "{'do_GET': lambda s: (s.send_response(200), s.end_headers(), "
        "s.wfile.write(b'Progress demo!'))})).serve_forever()",
    ],
    port=8000,
    ttl_seconds=300,
)

deployment = client.get(response.instance_name)
deployment.wait_until_ready(timeout=120, poll_interval=3, on_progress=on_progress)
```

Use async APIs for many short-lived deployments and cleanup all successful
deployments.

```python
import asyncio
from basilica import BasilicaClient

async def deploy_one(client: BasilicaClient, index: int):
    return await client.deploy_async(
        name=f"async-{index:02d}",
        source="app.py",
        env={"APP_ID": f"{index:02d}"},
        port=8000,
        ttl_seconds=180,
        timeout=180,
    )

async def main():
    client = BasilicaClient()
    deployments = await asyncio.gather(
        *(deploy_one(client, i) for i in range(1, 6)),
        return_exceptions=True,
    )

    ready = [d for d in deployments if not isinstance(d, Exception)]
    await asyncio.gather(*(d.delete_async() for d in ready), return_exceptions=True)

asyncio.run(main())
```

## Troubleshooting

- If `client.deploy()` times out, check `basilica deploy status <name>
  --show-phases` and `basilica deploy logs <name> --tail 100`.
- If the public URL returns 502/503, verify the process binds to `0.0.0.0` and
  the app port matches the deployment `port`.
- If a container fails immediately, verify it can run as UID 1000 and does not
  need root-only paths or privileged ports.
- If a Python-file deploy fails during startup, verify required packages are in
  `pip_packages` or `--pip`.
- If storage paths are missing, verify the mount path and wait for storage sync
  before treating it as an app bug.
- If GPU is unavailable, lower requirements or use `basilica ls` to inspect
  available capacity before retrying.
