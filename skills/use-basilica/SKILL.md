---
name: use-basilica
description: >
  Use when the user wants to operate Basilica from an AI coding tool,
  terminal, Python script, notebook, or CI job: CLI commands, Python SDK
  automation, authentication, credits and funding, GPU/CPU rentals,
  serverless deployments, inference endpoints, OpenClaw/Tau, distributed
  PyTorch or NCCL training, usage history, exact flags, safety checks, and
  cleanup guidance.
---

# Basilica

Use this skill to help users run Basilica as a customer cloud platform. Prefer
the shortest reliable control plane for the job, and keep cost-bearing actions
explicit.

## Control Plane Routing

- Use the CLI for interactive operator workflows: login, funding, discovery,
  rentals, SSH, copy/exec, serverless deploys, inference templates, share
  tokens, and cleanup.
- Use the Python SDK for repeatable automation, notebooks, CI, generated code,
  concurrent orchestration, programmatic deploys, usage history, and
  distributed PyTorch/NCCL training.
- Prefer direct rentals when the workload needs SSH, custom system setup,
  persistent rented hosts, manual model warmup, or very large models.
- Prefer serverless deploys when the user wants a public HTTP service,
  inference endpoint, hosted app URL, or short-lived demo.
- Use the CLI for deposit-address creation and deposit history. The SDK exposes
  balance and usage history, but not the full public deposit flow.

## Safety Rules

Treat these as cost-bearing actions and confirm intent unless the user already
asked to create resources:

```bash
basilica up ...
basilica deploy ...
basilica deploy vllm ...
basilica deploy sglang ...
basilica summon ...
basilica train up ...
```

SDK equivalents are also cost-bearing: `deploy()`, `deploy_vllm()`,
`deploy_sglang()`, `start_rental()`, `start_secure_cloud_rental()`,
`start_cpu_rental()`, `@basilica.deployment`, and `@basilica.distributed`.

Before creating resources, prefer read-mostly checks:

```bash
basilica balance
basilica ls
basilica ps
basilica deploy ls
basilica train ps
```

Use cleanup-friendly defaults:

- Add `--ttl` for CLI deploys and `ttl_seconds=` for SDK deploys unless the
  user explicitly wants persistence.
- State whether a resource will persist after the task.
- Tear down rentals with `basilica down <rental-id>` and deployments with
  `basilica deploy delete <name>` when finished unless the user asked to keep
  them.
- Use `basilica deploy delete`, not stale `basilica deployments delete`.

Deployments are public by default. Use `--private` in the CLI or `public=False`
in SDK deploys when access should be gated. OpenClaw deployments are
intentionally public and use their own gateway token flow.

## Install And Auth

Install the CLI:

```bash
curl -sSL https://basilica.ai/install.sh | bash
```

Interactive login:

```bash
basilica login
```

Headless shells, SSH boxes, containers, or remote terminals:

```bash
basilica login --device-code
```

Create an API token for SDK, CI, or notebooks:

```bash
basilica tokens create my-agent-token
export BASILICA_API_TOKEN="basilica_..."
```

The short alias `bs` is equivalent to `basilica` when installed by the official
installer.

## Output And Parsing

Use JSON for automation where the command supports it:

```bash
basilica --json balance
basilica --json ls
basilica --json ps
basilica --json deploy status my-app
basilica --json train ps
```

Use plain output for human-facing summaries. If a command lacks JSON or returns
a staged/unavailable feature, fall back to the SDK or a read command that
exposes the needed state.

## Account And Funding

```bash
basilica balance
basilica fund
basilica fund --tao
basilica fund --usd 25
basilica fund list --limit 100 --offset 0
basilica tokens create <name>
basilica tokens list
basilica tokens revoke <name> --yes
```

Use `basilica fund` for deposit address creation. Use `basilica fund list` for
deposit and card-funding history. Use SDK `get_balance()` and
`list_usage_history()` for programmatic balance and spend/usage checks.

## Rentals

### Discover Capacity

```bash
basilica ls
basilica ls h100
basilica ls --price-max 5 --country US
basilica ls --compute secure-cloud
basilica ls --compute community-cloud
basilica --json ls --compute citadel
```

### Start A Rental

For non-interactive rental startup, first select an explicit offering ID from
JSON discovery, then start the rental with `--offering-id`. Choose
`gpu_offerings[].id` or `cpu_offerings[].id`; spot vs on-demand is a property
of the selected offering. Do not combine `--offering-id` with positional GPU
filters, `--compute`, `--gpu-count`, `--spot`, `--region`, `--interconnect`, or
Bourse-only options.

```bash
basilica --json ls --compute citadel
basilica up --offering-id <offering-id> --name <name> --detach
```

Interactive or Bourse-style filtered rentals can use target filters:

```bash
basilica ssh-keys list
basilica ssh-keys add
basilica up h100 --gpu-count 1 --compute secure-cloud
```

### Operate A Rental

```bash
basilica ps
basilica status <rental-id>
basilica logs <rental-id> --tail 100
basilica ssh <rental-id>
basilica exec "nvidia-smi" --target <rental-id>
basilica cp ./local.txt <rental-id>:/workspace/local.txt
basilica restart <rental-id>
```

### Clean Up

```bash
basilica down <rental-id>
basilica down --all
```

### Volumes

Volumes are for secure-cloud rentals and must match provider and region:

```bash
basilica volumes create --name cache --size 100 --provider hyperstack --region US-1
basilica volumes attach cache --rental <rental-id>
basilica volumes list
basilica volumes detach cache --yes
basilica volumes delete cache --yes
```

### SDK Rental Automation

```python
from basilica import BasilicaClient

client = BasilicaClient()
key = client.get_ssh_key() or client.register_ssh_key("agent-key")
offerings = client.list_secure_cloud_gpus()
offering = sorted(offerings, key=lambda o: float(o.hourly_rate))[0]
rental = client.start_secure_cloud_rental(
    offering_id=offering.id,
    ssh_public_key_id=key.id,
)
print(rental.ssh_command)
```

## Serverless Deployments

CLI deploys:

```bash
basilica deploy my_api.py --name my-api --port 8000 --pip fastapi uvicorn --ttl 600
basilica deploy nginxinc/nginx-unprivileged:alpine --name nginx-demo --port 8080 --ttl 300
basilica deploy inference.py --name gpu-model --gpu 1 --gpu-model H100 --memory 32Gi --pip torch --ttl 3600
basilica deploy hello.py --name stateful-app --storage --storage-path /data --ttl 3600
```

Manage deploys:

```bash
basilica deploy ls
basilica deploy status <name> --show-phases
basilica deploy logs <name> --tail 100
basilica deploy logs <name> --follow
basilica deploy scale <name> --replicas 3
basilica deploy restart <name>
basilica deploy delete <name> --yes
```

Private deployments:

```bash
basilica deploy my_api.py --name private-app --port 8000 --private --ttl 600
basilica deploy share-token status private-app
basilica deploy share-token regenerate private-app
basilica deploy share-token revoke private-app --yes
```

SDK high-level deploy:

```python
from basilica import BasilicaClient

client = BasilicaClient()
deployment = client.deploy(
    name="hello-api",
    source="app.py",
    port=8000,
    pip_packages=["fastapi", "uvicorn"],
    ttl_seconds=600,
)
print(deployment.url)
print(deployment.logs(tail=100))
deployment.delete()
```

SDK decorator deploy:

```python
import basilica

@basilica.deployment(
    name="hello-fastapi",
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
        return {"status": "ok"}

    uvicorn.run(app, host="0.0.0.0", port=8000)

deployment = serve()
print(deployment.url)
```

SDK deployment notes:

- `deploy()` blocks until readiness and returns a `Deployment` with `url`,
  `status()`, `logs()`, `refresh()`, and `delete()`.
- Use `deploy_async()` and async methods for concurrent orchestration.
- Use `create_deployment()` only when the high-level `deploy()` surface lacks
  needed control.
- For failure details, `client.get_deployment(name).message` may be more direct
  than the high-level wrapper.

## Inference Templates

CLI:

```bash
basilica deploy vllm Qwen/Qwen2.5-0.5B-Instruct --name my-vllm --ttl 3600
basilica deploy sglang Qwen/Qwen2.5-0.5B-Instruct --name my-sglang --ttl 3600
```

SDK:

```python
from basilica import BasilicaClient

client = BasilicaClient()
deployment = client.deploy_vllm(
    model="Qwen/Qwen2.5-0.5B-Instruct",
    name="my-vllm",
    ttl_seconds=3600,
)
print(f"{deployment.url}/v1/chat/completions")
```

For very large models, rentals may be a better first choice when the workload
needs manual control, custom setup, or warmup longer than deployment health
checks tolerate.

## OpenClaw And Tau

```bash
basilica summon openclaw --provider openai
basilica summon openclaw --provider anthropic
basilica summon tau
```

OpenClaw deploys are intentionally public. Access is controlled by the OpenClaw
gateway token, not Basilica share-token auth.

## Distributed Training

For Python DDP, DiLoCo, FSDP, or NCCL collective workloads, prefer the
`@basilica.distributed` SDK surface. The same entry point also supports BYO
launchers with `command=[...]`.

Decorator mode:

```python
import basilica
from basilica import ProviderFilter, WorldSize

@basilica.distributed(
    name="dlc-demo",
    image="ghcr.io/one-covenant/basilica/basilica-distributed-trainer:latest",
    world_size=WorldSize(min=2, target=4, max=4),
    gpu_count=1,
    gpu_models=["A100"],
    provider_filter=ProviderFilter(include=["hyperstack", "verda"]),
    topology_spread="pack",
    bench=True,
)
def train():
    import os
    import torch.distributed as dist

    dist.init_process_group(backend="nccl")
    print(os.environ["RANK"], os.environ["WORLD_SIZE"], os.environ["LOCAL_RANK"])
    dist.destroy_process_group()

with train() as training:
    training.wait_until_complete(timeout=1800)
    print(training.bench)
```

BYO launcher mode:

```python
import basilica
from basilica import ProviderFilter, WorldSize

training = basilica.distributed(
    name="dlc-torchrun",
    image="ghcr.io/one-covenant/basilica/basilica-distributed-trainer:latest",
    command=[
        "torchrun",
        "--rdzv-backend=etcd",
        "--rdzv-endpoint=$BASILICA_RDZV_ENDPOINT",
        "--rdzv-id=$BASILICA_RDZV_ID",
        "--nnodes=$BASILICA_WORLD_TARGET",
        "--nproc-per-node=$BASILICA_GPUS_PER_POD",
        "/workspace/train.py",
    ],
    world_size=WorldSize(min=2, target=2, max=4),
    gpu_count=1,
    gpu_models=["A100"],
    provider_filter=ProviderFilter(include=["hyperstack", "verda"]),
    topology_spread="pack",
)

with training:
    training.scale(target=3)
    training.wait_until_complete(timeout=1800)
```

Distributed rules:

- Use `with training:` for mid-run orchestration and auto-cleanup.
- `bench=True` opts in to the per-job NCCL bandwidth probe. Read the result via
  `training.bench`; `None` means no measurement.
- Do not use removed legacy SDK symbols:
  `client.deploy_distributed_managed(...)`, `bench="on-start"`,
  `training.bench_status`, or `training.wait_until_bench_complete()`.
- CLI `basilica train` is available for command-launched distributed jobs:

```bash
basilica train up \
  dlc-demo \
  --image ghcr.io/one-covenant/basilica/basilica-distributed-trainer:latest \
  --command 'torchrun --rdzv-backend=etcd --rdzv-endpoint=$BASILICA_RDZV_ENDPOINT --rdzv-id=$BASILICA_RDZV_ID --nnodes=$BASILICA_WORLD_TARGET --nproc-per-node=$BASILICA_GPUS_PER_POD /workspace/train.py' \
  --world-size 2:4:4 \
  --gpu-count 1 \
  --gpu-model A100 \
  --provider hyperstack \
  --provider verda \
  --topology-spread pack \
  --bench on-start \
  --ttl-seconds 3600

basilica train ps
basilica train ls
basilica train logs <name> --tail 100
basilica train events <name>
basilica train bench <name>
basilica train scale <name> --target 3
basilica train down <name>
```

## SDK Utilities

```python
from basilica import BasilicaClient

client = BasilicaClient()
print(client.health_check().status)
print(client.get_balance())
print(client.list_usage_history(limit=20, offset=0))

for deployment in client.list():
    print(deployment.name, deployment.state)
```

## Agent Skills Maintenance

Install or update Basilica agent skills:

```bash
basilica skills install
basilica skills install -y
basilica skills install --agent codex
basilica skills list
basilica skills uninstall --agent codex
```

`basilica skills install` installs public user-facing skills from the Basilica
skills repository. It does not install internal developer-only repo skills.
