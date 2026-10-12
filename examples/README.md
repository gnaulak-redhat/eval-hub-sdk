# How to run the examples

## OCI job exports

[oci_job.yaml](oci_job.yaml) demonstrates CLI usage with a local mode server,
Ollama running locally with the model `llama3.2:3b-instruct-q4_K_M` at
`http://localhost:11434/v1`, and an anonymous OCI registry at
`http://localhost:5001`. The CLI defaults to `http://localhost:8080`.
The example limits `gsm8k` to five examples to reduce runtime. Replace the
provider ID if your configured provider supporting `gsm8k` has a different ID.

```yaml
model:
  url: http://localhost:11434/v1
  name: llama3.2:3b-instruct-q4_K_M
benchmarks:
  - id: gsm8k
    provider_id: 5f1ee53f-35e6-44ca-bc6a-d4c28e2c0f97
    parameters:
      num_examples: 5
```

```shell
evalhub eval run --config examples/oci_job.yaml
evalhub eval status <job-id>
```

Evaluation waits can finish before OCI publication finishes. Run
`evalhub eval status <job-id>` again to check the OCI state. An OCI publication
failure does not change a completed evaluation into a failed evaluation.
Current OCI completion covers evaluation-card publication; bundle publication
and signing are future work.

Human-readable status shows only the OCI state; results omit OCI reporting.
Use `evalhub eval status <job-id> --format json` (or `--format yaml`) for
complete OCI information, including messages and published artifact references.
Additional OCI fields in human-readable status or results can be considered
in a future change if user feedback requires them.

## Watching logs

**File:** [watch_job_logs.py](watch_job_logs.py)

The script streams new log lines to stdout and prints the final job state to
stderr when the job reaches a terminal state.

### Exit codes

| Code | Meaning |
| ---- | ------- |
| `0` | Job reached a terminal state (completed, cancelled, partially_failed) |
| `1` | Job finished `failed` |
| `2` | Watch ended before completion (timeout or non-terminal last state) |
| `3` | HTTP/request error |
| `130` | Interrupted with Ctrl+C |

### 1. Configure connection (optional, reusable)

**Default profile** (used when `--profile` is omitted):

```shell
evalhub config set base_url https://evalhub.apps.my-cluster.example.com
evalhub config set token "$(oc whoami -t)"
evalhub config set tenant my-team
```

**Named profile** (use with `--profile prod`):

```shell
evalhub config set base_url https://evalhub.apps.my-cluster.example.com --profile prod
evalhub config set token "$(oc whoami -t)" --profile prod
evalhub config set tenant my-team --profile prod
```

### 2. Get a job ID

Submit a job or list existing jobs:

```shell
uv run evalhub eval run --config eval.yaml
# → Job submitted: 86c904cc-cdcf-452d-b9d8-754a7d6391ec

uv run evalhub eval status --status running
```

### 3. Watch logs

**Default profile** (reads `~/.config/evalhub/config.yaml`):

```shell
cd eval-hub-sdk
uv run python examples/watch_job_logs.py 86c904cc-cdcf-452d-b9d8-754a7d6391ec
```

**Named profile:**

```shell
uv run python examples/watch_job_logs.py 86c904cc-cdcf-452d-b9d8-754a7d6391ec --profile prod
```

**Explicit remote settings** (no profile file required):

```shell
export EVALHUB_BASE_URL=https://evalhub.apps.my-cluster.example.com
export EVALHUB_TOKEN="$(oc whoami -t)"
export EVALHUB_TENANT=my-team

uv run python examples/watch_job_logs.py 86c904cc-cdcf-452d-b9d8-754a7d6391ec
```

Or pass flags directly:

```shell
uv run python examples/watch_job_logs.py 86c904cc-cdcf-452d-b9d8-754a7d6391ec \
  --base-url https://evalhub.apps.my-cluster.example.com \
  --token "$(oc whoami -t)" \
  --tenant my-team
```

**Inside a cluster** (service account token):

```shell
uv run python examples/watch_job_logs.py 86c904cc-cdcf-452d-b9d8-754a7d6391ec \
  --base-url https://evalhub.evalhub.svc:8080 \
  --token-file /var/run/secrets/kubernetes.io/serviceaccount/token \
  --ca-bundle /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  --tenant default
```

### Useful flags

```shell
uv run python examples/watch_job_logs.py --help
```

Common options:

- `--benchmark-index 0` — watch a single benchmark only
- `--tail-lines 500` — max lines returned per poll (default: 1000)
- `--poll-interval 3` — seconds between polls (default: 2.0)
- `--timeout 3600` — stop watching after one hour
- `--timestamps` — include Kubernetes log timestamps when supported
