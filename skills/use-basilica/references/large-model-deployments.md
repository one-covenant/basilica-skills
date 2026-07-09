# Large Model Deployments

Use this reference when a user wants to deploy a large vLLM or SGLang model, or
when an inference deployment is failing during download, startup, health checks,
or model loading.

## Scope

This reference is for hosted HTTP inference deployment flows.

## Deployment Ladder

Start with the highest-level surface that gives enough control:

1. Use `client.deploy_vllm()` or `client.deploy_sglang()` for ordinary inference
   servers. These helpers cover common GPU sizing, model caching, health checks,
   and OpenAI-compatible endpoints.
2. Use `client.deploy(...)` or `@basilica.deployment` when the server needs
   custom source, explicit images, custom env vars, explicit probe settings,
   retries, or a longer timeout.
3. Use `CreateDeploymentRequest` when the model needs precise resources,
   low-level command/args, multi-GPU tensor parallelism, custom vLLM/SGLang
   images, or explicit `HealthCheckConfig` that the helper surface cannot
   express clearly.

Keep cost-bearing behavior explicit. Use `ttl_seconds=` for experiments and
show cleanup with `deployment.delete()` or `basilica deploy delete <name>`.

## Large Model Knobs

Set these deliberately for large or slow-loading models:

- GPU shape: `gpu_count`, GPU model, and per-GPU VRAM floor such as
  `min_gpu_memory_gb`.
- Container resources: CPU and system RAM such as `cpu="32"` and
  `memory="512Gi"` for 70B/1T-class model servers.
- Runtime image: use a vLLM/SGLang image that supports the model architecture;
  use a custom image when stable upstream images do not include required model
  classes or parsers.
- Server args: tensor parallelism, context length, `trust_remote_code`,
  tool-call parser, reasoning parser, dtype, GPU memory utilization, and any
  model-specific flags from the model vendor.
- Startup budget: set deployment `timeout` plus startup/readiness/liveness
  probes to cover download, shard loading, CUDA graph capture, and first health
  response.
- Download reliability: set Hugging Face timeout/cache env vars or pre-download
  with retry logic when model files are large or flaky.
- Observability: tell the user to follow `basilica deploy logs <name> --follow`
  during startup; a deployment can be healthy eventually even if the initial
  client wait times out.

## Template Helper Example

Use this for common vLLM or SGLang models before reaching for low-level APIs.

```python
import basilica

client = basilica.BasilicaClient()

deployment = client.deploy_vllm(
    model="meta-llama/Llama-2-7b-hf",
    name="llama2-7b-server",
    gpu_count=1,
    memory="32Gi",
    dtype="float16",
    trust_remote_code=True,
    ttl_seconds=3600,
)

print(deployment.url)
print(f"{deployment.url}/v1/chat/completions")
deployment.delete()
```

For SGLang, use the SGLang helper and tune SGLang-specific options:

```python
import basilica

client = basilica.BasilicaClient()

deployment = client.deploy_sglang(
    model="Qwen/Qwen2.5-3B-Instruct",
    name="qwen-3b-sglang",
    gpu_count=1,
    context_length=8192,
    mem_fraction_static=0.85,
    trust_remote_code=True,
    ttl_seconds=3600,
)

print(deployment.url)
print(f"{deployment.url}/v1/chat/completions")
deployment.delete()
```

## Custom Health Checks For Slow Startup

Large models can be killed before they finish loading if the startup window is
too short. Configure a startup probe first; liveness and readiness should not
drive restarts until startup has had enough time.

```python
from basilica import BasilicaClient, HealthCheckConfig, ProbeConfig

PORT = 8000

def startup_health_check(startup_minutes: int) -> HealthCheckConfig:
    initial_delay = 480
    period = 120
    timeout = 120
    failures = max(1, (startup_minutes * 60 - initial_delay) // period)

    return HealthCheckConfig(
        startup=ProbeConfig(
            path="/health",
            port=PORT,
            initial_delay_seconds=initial_delay,
            period_seconds=period,
            timeout_seconds=timeout,
            failure_threshold=failures,
        ),
        liveness=ProbeConfig(
            path="/health",
            port=PORT,
            initial_delay_seconds=initial_delay,
            period_seconds=period,
            timeout_seconds=timeout,
            failure_threshold=5,
        ),
        readiness=ProbeConfig(
            path="/health",
            port=PORT,
            initial_delay_seconds=initial_delay,
            period_seconds=period,
            timeout_seconds=timeout,
            failure_threshold=5,
        ),
    )

client = BasilicaClient()
health_check = startup_health_check(startup_minutes=45)

deployment = client.deploy(
    name="sglang-large-model",
    source="server.py",
    image="lmsysorg/sglang:latest",
    port=PORT,
    health_check=health_check,
    timeout=46 * 60,
    ttl_seconds=3600,
    gpu_count=1,
    gpu_models=["A100"],
    min_gpu_memory_gb=80,
    cpu="2",
    memory="64Gi",
    env={
        "HF_HUB_DISABLE_SYMLINKS_WARNING": "1",
        "HF_HUB_DISABLE_XET": "1",
    },
)

print(f"logs: basilica deploy logs {deployment.name} --follow")
```

## Low-Level Multi-GPU vLLM Example

Use this shape for 70B/1T-class models that need exact tensor parallelism,
high-VRAM GPUs, custom images, long startup budgets, or model-specific parsers.

```python
import basilica
from basilica import (
    BasilicaClient,
    CreateDeploymentRequest,
    GpuRequirementsSpec,
    HealthCheckConfig,
    ProbeConfig,
    ResourceRequirements,
)

client = BasilicaClient()
model = "moonshotai/Kimi-K2-Instruct"

args = [
    "serve",
    model,
    "--host",
    "0.0.0.0",
    "--port",
    "8000",
    "--tensor-parallel-size",
    "8",
    "--trust-remote-code",
    "--tool-call-parser",
    "kimi_k2",
    "--enable-auto-tool-choice",
    "--max-model-len",
    "32768",
    "--gpu-memory-utilization",
    "0.95",
]

resources = ResourceRequirements(
    cpu="32",
    memory="512Gi",
    gpus=GpuRequirementsSpec(
        count=8,
        model=["H200"],
        min_cuda_version=None,
        min_gpu_memory_gb=80,
    ),
)

health_check = HealthCheckConfig(
    liveness=ProbeConfig(
        path="/health",
        port=8000,
        initial_delay_seconds=5400,
        period_seconds=30,
        timeout_seconds=10,
        failure_threshold=3,
    ),
    readiness=ProbeConfig(
        path="/health",
        port=8000,
        initial_delay_seconds=5400,
        period_seconds=10,
        timeout_seconds=5,
        failure_threshold=3,
    ),
)

request = CreateDeploymentRequest(
    instance_name="kimi-k2-instruct",
    image="vllm/vllm-openai:latest",
    replicas=1,
    port=8000,
    command=["vllm"],
    args=args,
    env={"HF_HUB_DOWNLOAD_TIMEOUT": "3600"},
    resources=resources,
    ttl_seconds=7200,
    public=True,
    storage=None,
    health_check=health_check,
)

response = client._client.create_deployment(request)
deployment = basilica.Deployment._from_response(client, response)

try:
    deployment.wait_until_ready(timeout=2400, silent=False)
except basilica.exceptions.DeploymentFailed:
    print(f"Still loading. Follow logs: basilica deploy logs {deployment.name} --follow")

print(f"{deployment.url}/v1/chat/completions")
```

For newer model architectures not available in the stable runtime image, keep
the same deployment shape but switch to a custom image that contains the needed
vLLM/SGLang build, parser, or model class.

## Failure Handling

When startup fails, inspect in this order:

1. `basilica deploy status <name> --show-phases`
2. `basilica deploy logs <name> --tail 100`
3. `basilica deploy logs <name> --follow`
4. Increase startup probe window and client timeout if logs show active
   download/loading rather than a crash.
5. Change image or server args if logs show missing model architecture, parser,
   CUDA/runtime incompatibility, or unsupported model flags.
6. Increase GPU count, GPU memory floor, CPU, or system RAM if logs show
   out-of-memory, scheduler mismatch, or tensor-parallel placement failures.

If the deployment is only still loading, do not delete it reflexively. Tell the
user how to monitor logs and how to delete it if they do not want to keep paying
for the experiment.
