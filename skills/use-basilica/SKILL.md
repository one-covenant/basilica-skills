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

## Important Rules

### Rental Startup

Follow this rule before starting rentals. First list offerings with JSON
discovery, select an explicit offering ID, then start the rental with
`--offering-id`.

Choose `gpu_offerings[].id` or `cpu_offerings[].id`; spot vs on-demand is a
property of the selected offering. Do not combine `--offering-id` with
positional GPU filters, `--compute`, `--gpu-count`, `--spot`, `--region`,
`--interconnect`, or Bourse-only options.

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
basilica deploy openclaw ...
basilica deploy tau ...
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

Use `basilica fund` for deposit address creation. Use `basilica fund list` for
deposit and card-funding history. Use SDK `get_balance()` and
`list_usage_history()` for programmatic balance and spend/usage checks.

For exact flags, run:

```bash
basilica balance --help
basilica fund --help
basilica fund list --help
basilica tokens --help
```

## Rentals

### Discover Capacity

Use `basilica ls` for capacity discovery. Use JSON when selecting an offering
for automation, and prefer explicit `--compute citadel` or `--compute bourse`
when the source matters.

```bash
basilica --json ls --compute citadel
```

For filters and output options, run:

```bash
basilica ls --help
```

### Start A Rental

```bash
basilica --json ls --compute citadel
basilica up --offering-id <offering-id> --name <name> --detach
```

Interactive or Bourse-style filtered rentals can use target filters:

```bash
basilica ssh-keys list
basilica ssh-keys add
basilica up h100 --gpu-count 1 --compute bourse
```

For exact startup flags, run:

```bash
basilica up --help
basilica ssh-keys --help
```

### Operate A Rental

Use `basilica ps` to find active rentals before operating on one. For exact
flags, run the command-specific help:

```bash
basilica ps --help
basilica status --help
basilica logs --help
basilica ssh --help
basilica exec --help
basilica cp --help
basilica restart --help
```

### Clean Up And Volumes

Use `basilica down <rental-id>` for rental cleanup. Volumes must match provider
and region, and should be deleted when no longer needed.

```bash
basilica down --help
basilica volumes --help
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

Use CLI deploys for quick services, hosted URLs, container images, and
inference-style endpoints. Add `--ttl` unless the user explicitly wants the
deployment to persist.

```bash
basilica deploy my_api.py --name my-api --port 8000 --pip fastapi uvicorn --ttl 600
```

For exact deployment, management, private access, storage, GPU, and health-check
flags, run:

```bash
basilica deploy --help
basilica deploy ls --help
basilica deploy status --help
basilica deploy logs --help
basilica deploy scale --help
basilica deploy restart --help
basilica deploy delete --help
basilica deploy share-token --help
```

Private deployments require `--private`; share tokens are managed under
`basilica deploy share-token`.

```bash
basilica deploy my_api.py --name private-app --port 8000 --private --ttl 600
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

SDK deployment notes:

- `deploy()` blocks until readiness and returns a `Deployment` with `url`,
  `status()`, `logs()`, `refresh()`, and `delete()`.
- Use `deploy_async()` and async methods for concurrent orchestration.
- Use `@basilica.deployment(...)` when a decorator interface fits the calling
  code better than `client.deploy(...)`.
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

For exact model server flags, run:

```bash
basilica deploy vllm --help
basilica deploy sglang --help
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
basilica deploy openclaw --provider openai --ttl 3600
basilica deploy openclaw --provider anthropic --ttl 3600
basilica deploy tau --ttl 3600
```

For exact gateway and agent flags, run:

```bash
basilica deploy openclaw --help
basilica deploy tau --help
```

OpenClaw deploys are intentionally public. Access is controlled by the OpenClaw
gateway token, not Basilica share-token auth.

## Distributed Training

For Python DDP, DiLoCo, FSDP, or NCCL collective workloads, prefer the
`@basilica.distributed` SDK surface. The same entry point also supports BYO
launchers with `command=[...]`.

Distributed rules:

- Use `with training:` for mid-run orchestration and auto-cleanup.
- `bench=True` opts in to the per-job NCCL bandwidth probe. Read the result via
  `training.bench`; `None` means no measurement.
- Do not use removed legacy SDK symbols:
  `client.deploy_distributed_managed(...)`, `bench="on-start"`,
  `training.bench_status`, or `training.wait_until_bench_complete()`.
- Use CLI `basilica train` for command-launched distributed jobs.
- CLI-side launches are BYO-launcher only; local Python source shipping is an
  SDK decorator feature.

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

For exact training flags and lifecycle commands, run:

```bash
basilica train --help
basilica train up --help
basilica train logs --help
basilica train bench --help
basilica train down --help
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
```

`basilica skills install` installs public user-facing skills from the Basilica
skills repository. It does not install internal developer-only repo skills.
For exact install targets and confirmation flags, run:

```bash
basilica skills --help
basilica skills install --help
basilica skills uninstall --help
```
