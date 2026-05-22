---
name: kv-cache-offloading-onboarding
description: Understanding the architecture before running the smoke test and AgentBench
metadata:
  type: documentation
---

# KV Cache Offloading Onboarding Guide

## Context

This project provides a reproducible harness for proving that AgentBench hints flow through the Dynamo frontend → SGLang worker pipeline. The goal is to run SWE-bench benchmarks and verify that worker runtime logs contain `[RUNTIME_JSON]` events with `agent_hints`, `hint_probe_id`, and `request_context`.

You're being onboarded to eventually run this on the **Hopper** machine (gracehopper), but first need to understand the architecture before blindly running commands.

---

## Architecture Overview

### The Big Picture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         LOCAL DEVELOPMENT                             │
│  (You build & test here before uploading to Hopper)                  │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  │ upload.sh (rsync via jump host)
                                  ↓
┌──────────────────────────────────────────────────────────────────────┐
│                      HOPPER MACHINE (gracehopper)                     │
│  Machine: ojaiyeob@gracehopper (via falcon.7elements.com:1337)       │
│  Location: /home/central/ojaiyeob/kv_cache_offloading-main/          │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │              Dynamo Frontend (Docker Container)                │  │
│  │  Image: local/dynamo-frontend:runtime-json-logs                │  │
│  │  Port: 8000 (OpenAI-compatible API)                            │  │
│  │                                                                 │  │
│  │  [BLUE CODE LAYER #1: Rust Preprocessor]                       │  │
│  │  - Extracts nvext.agent_hints from HTTP requests               │  │
│  │  - Packages them into runtime_observability                    │  │
│  │  - Forwards to worker via extra_args                           │  │
│  └─────────────────────────┬──────────────────────────────────────┘  │
│                            │                                          │
│                            │ (internal network)                       │
│                            ↓                                          │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │               Dynamo SGLang Worker (Docker Container)          │  │
│  │  Image: local/dynamo-sglang:runtime-json-logs                  │  │
│  │  GPU: All GPUs (--gpus all)                                    │  │
│  │  Model: Qwen/Qwen2.5-7B-Instruct (default)                     │  │
│  │                                                                 │  │
│  │  [BLUE CODE LAYER #2: Python Runtime Logging]                  │  │
│  │  - Emits [RUNTIME_JSON] structured logs                        │  │
│  │  - Contains agent_hints, hint_probe_id, request_context        │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  etcd (Service Registry) - Port 2379                           │  │
│  │  NATS (Message Broker) - Port 4222                             │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
                                  ↑
                                  │ HTTP POST to :8000/v1/chat/completions
                                  │
                    ┌─────────────────────────────┐
                    │   AgentBench Runner         │
                    │  (deepagents_swebench...)   │
                    │                             │
                    │  Constructs nvext payload:  │
                    │  {                          │
                    │    agent_hints: {...},      │
                    │    request_context: {...}   │
                    │  }                          │
                    └─────────────────────────────┘
```

---

## What's What, What's Where

### 1. The "Blue Code" (The Instrumentation)

This is the custom code that makes hints visible in worker logs. It consists of **two patches**:

#### Blue Layer #1: Frontend Hint Preservation (Rust)
- **File**: `runtime_instrumentation/patches/dynamo_preserve_agent_hints_to_worker.patch`
- **What it does**: Modifies Dynamo's Rust preprocessor to extract `nvext.agent_hints` from incoming HTTP requests and forward them to the worker
- **Key function**: `runtime_observability_extra_args_from_nvext()`

#### Blue Layer #2: Runtime JSON Logging (Python)
- **File**: `runtime_instrumentation/patches/dynamo_runtime_json_logging.patch`
- **What it does**: Adds structured logging to both frontend and worker, emitting `[RUNTIME_JSON]` events
- **Key module**: `components/src/dynamo/common/runtime_logging.py`
- **Example output**: `[RUNTIME_JSON] {"worker.decode.request_received": {..., "hint_probe_id": "abc123::hint_probe"}}`

### 2. The Docker Containers

#### Published Default Images (For Smoke Test)
- `nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.0.2` (both frontend & worker)
- **Use for**: Quick validation that Docker, GPU, and model loading work
- **Does NOT contain**: The blue code instrumentation

#### Custom Instrumented Images (For Real Benchmark Runs)
- `local/dynamo-frontend:runtime-json-logs`
- `local/dynamo-sglang:runtime-json-logs`
- **Contains**: Both blue code layers applied
- **Build with**: `runtime_instrumentation/build_instrumented_dynamo_images.sh`

### 3. The Benchmark (The "First Row")

- **File**: `agentbench/deepagents_swebench_single_host.py`
- **What it does**: Runs SWE-bench Pro tasks through the Dynamo frontend
- **The "first row"**: Refers to task index 0 (first SWE-bench task) - the minimal smoke test
- **Constructs hints**: In `agentbench/deepagents_app/src/agent.py` via `build_phase_hints()` and `build_dynamo_chat_model()`
- **Output**: Creates timestamped result directories like `agentbench/results/agentbench-nodebb_20260519_140124/`

### 4. The Upload Script

- **File**: `upload.sh`
- **Target**: `ojaiyeob@gracehopper:/home/central/ojaiyeob/kv_cache_offloading-main/`
- **Jump Host**: `falcon.7elements.com:1337`
- **Method**: rsync over SSH with `--proxy-jump`
- **Dry Run**: `./upload.sh --dry` to preview what would be uploaded
- **Excludes**: `.git/`, `__pycache__/`, `.venv/`, Docker artifacts, etc.

### 5. Machine Configuration Files

The README doesn't contain explicit "machine config files" for Hopper, but the key configuration happens through:
- Environment variables in the run scripts (see `run_dynamo_single_host.sh`)
- Docker compose-like orchestration via the three run scripts

---

## The Onboarding Flow (Step-by-Step)

### Step 0: Prerequisites on Hopper
- Python 3.11
- Docker with NVIDIA GPU access
- 80-120 GB free disk space
- `HF_TOKEN` environment variable set

### Step 1: Smoke Test (No Rebuild)
**Goal**: Prove Docker, GPU, and model loading work

```bash
cd ~/kv_cache_offloading
./run_dynamo_single_host.sh start  # Uses published images
./run_dynamo_single_host.sh test   # Quick validation
```

**What this validates**:
- ✅ Docker containers start
- ✅ GPU is accessible
- ✅ Model loads
- ✅ OpenAI-compatible API responds
- ❌ Does NOT prove hints reach worker logs (need custom images for that)

### Step 2: Build Instrumented Images
**Goal**: Create custom Docker images with the blue code patches

```bash
cd ~/kv_cache_offloading

# Prepare source (fetch upstream + apply patches)
./runtime_instrumentation/prepare_instrumented_dynamo_source.sh

# Build custom images
LEAN_FRONTEND=1 DYN_RUNTIME_JSON_LOGS=1 \
./runtime_instrumentation/build_instrumented_dynamo_images.sh
```

**What this creates**:
- `local/dynamo-frontend:runtime-json-logs`
- `local/dynamo-sglang:runtime-json-logs`

### Step 3: Start Instrumented Runtime
**Goal**: Run Dynamo with hint logging enabled

```bash
./run_dynamo_single_host.sh stop  # Clean slate

DYN_RUNTIME_JSON_LOGS=1 \
DYN_TOOL_CALL_PARSER=hermes \
DYNAMO_MODEL_PATH='Qwen/Qwen2.5-7B-Instruct' \
DYNAMO_SERVED_MODEL_NAME='Qwen/Qwen2.5-7B-Instruct' \
FRONTEND_IMAGE=local/dynamo-frontend:runtime-json-logs \
WORKER_IMAGE=local/dynamo-sglang:runtime-json-logs \
./run_dynamo_single_host.sh start
```

**What this does**:
- Starts etcd, NATS, frontend, and worker containers
- Enables runtime JSON logging via `DYN_RUNTIME_JSON_LOGS=1`
- Uses the custom instrumented images

### Step 4: Run AgentBench (The Real Test)
**Goal**: Prove hints flow through the entire stack

```bash
cd ~/kv_cache_offloading

python3.11 agentbench/deepagents_swebench_single_host.py \
  --app-variant upstream_deploy_coding_agent \
  --frontend-url http://127.0.0.1:8000/v1/chat/completions \
  --model Qwen/Qwen2.5-7B-Instruct \
  --dataset ScaleAI/SWE-bench_Pro \
  --split test \
  --index 0 \
  --prompt-evolution-value-char-limit 1000
```

**What this does**:
- Runs first SWE-bench task (index 0)
- Sends requests with `nvext.agent_hints` to Dynamo
- Collects worker logs and runtime events
- Creates result directory with analysis

### Step 5: Verify Results
**Goal**: Confirm hints reached the worker logs

```bash
LATEST_RESULT="$(ls -td agentbench/results/* | head -1)"

# Check for hint presence
grep -R "hint_probe_id\|agent_hints\|worker.decode" -n "$LATEST_RESULT" | head -50

# Read analysis reports
cat "$LATEST_RESULT/runtime_hint_alignment_analysis.md"
cat "$LATEST_RESULT/others/runtime_hint_alignment_summary_table.csv"
```

**Success signal**:
- ✅ `others/worker_runtime.log` contains `[RUNTIME_JSON]` events
- ✅ Events include `hint_probe_id: "...::hint_probe"`
- ✅ `agent_hints` and `request_context` are present

---

## Docker Orchestration Deep Dive

The three run scripts work together like a mini-orchestrator. Here's how they coordinate:

### Script Hierarchy

```
run_dynamo_single_host.sh (Orchestrator)
    │
    ├─→ run_dynamo_head.sh (Frontend + Services)
    │   ├─→ Starts dynamo-etcd container
    │   ├─→ Starts dynamo-nats container
    │   └─→ Starts dynamo-frontend container
    │
    └─→ run_dynamo_worker.sh (Worker)
        └─→ Starts dynamo-sglang-worker container
```

### How `run_dynamo_single_host.sh` Works

**Key Responsibilities**:
1. **Propagate environment variables** - Passes all config vars to both head and worker scripts
2. **Wait for model registration** - Polls `/v1/models` endpoint until the served model appears
3. **Provide unified commands** - `status`, `logs`, `test` work across both head and worker

**Critical Flow** (when you run `./run_dynamo_single_host.sh start`):
```bash
# Step 1: Start head (frontend + services)
./run_dynamo_head.sh start
  ├─→ Start etcd container (port 2379)
  ├─→ Wait for etcd to be running
  ├─→ Start NATS container (port 4222)
  ├─→ Wait for NATS to be running
  ├─→ Start frontend container (port 8000)
  ├─→ Wait for frontend container to be running
  └─→ Wait for frontend /health endpoint to respond

# Step 2: Start worker
./run_dynamo_worker.sh start
  ├─→ Check GPU compatibility (rejects T4, requires Ampere+)
  ├─→ Derive NATS endpoint from ETCD_ENDPOINTS if not set
  ├─→ Create cache directory (/mnt/docker-data/dynamo_cache)
  ├─→ Start worker container with --gpus all
  └─→ Wait 3 seconds and verify container is still running

# Step 3: Wait for model registration (single_host only)
wait_for_model_registration()
  ├─→ Poll http://127.0.0.1:8000/v1/models every 2 seconds
  ├─→ Up to 120 retries (4 minutes total)
  └─→ Look for {"id":"Qwen/Qwen2.5-7B-Instruct"} in response
```

### Container Network Architecture

All containers use `--network host`, meaning they share the host's network namespace:

| Container | Listens On | Connects To |
|-----------|------------|-------------|
| `dynamo-etcd` | `:2379` | (none - service registry) |
| `dynamo-nats` | `:4222` | (none - message broker) |
| `dynamo-frontend` | `:8000` (HTTP) | etcd @ `:2379`, NATS @ `:4222` |
| `dynamo-sglang-worker` | (internal) | etcd @ `:2379`, NATS @ `:4222` |

**Why `--network host`?**
- Simplifies discovery (no need for Docker DNS or bridge networks)
- Frontend and worker find etcd/NATS at `localhost:2379` and `localhost:4222`
- Frontend HTTP API is directly accessible at `127.0.0.1:8000`

### Volume Mounts & State Persistence

#### Head State (Frontend)
- **Host Path**: `~/kv_cache_offloading/dynamo_head_state/`
- **Containers**:
  - `dynamo-etcd`: `-v ~/kv_cache_offloading/dynamo_head_state/etcd-data:/etcd-data`
    - Persists service registry state across restarts
  - `dynamo-frontend`: No volumes (stateless except for etcd registration)

#### Worker State
- **Host Path**: `/mnt/docker-data/dynamo_cache/`
- **Container**: `dynamo-sglang-worker`
  - `-v /mnt/docker-data/dynamo_cache:/models/hfcache`
  - **Purpose**: Hugging Face model cache (Qwen downloads go here)
  - **Why /mnt/docker-data?** Intended for large persistent volumes on cloud machines

### Environment Variable Propagation

When you run:
```bash
DYN_RUNTIME_JSON_LOGS=1 \
FRONTEND_IMAGE=local/dynamo-frontend:runtime-json-logs \
WORKER_IMAGE=local/dynamo-sglang:runtime-json-logs \
./run_dynamo_single_host.sh start
```

Here's how variables flow:

```
run_dynamo_single_host.sh
  ├─→ Reads: DYN_RUNTIME_JSON_LOGS, FRONTEND_IMAGE, WORKER_IMAGE
  │
  ├─→ Passes to run_dynamo_head.sh:
  │   ├─ ETCD_ENDPOINTS=http://127.0.0.1:2379
  │   ├─ NATS_SERVER=nats://127.0.0.1:4222
  │   ├─ DYN_RUNTIME_JSON_LOGS=1
  │   └─ DYNAMO_FRONTEND_PORT=8000
  │       ↓
  │   dynamo-frontend container gets:
  │       -e DYN_RUNTIME_JSON_LOGS=1
  │       -e ETCD_ENDPOINTS=http://127.0.0.1:2379
  │       -e NATS_SERVER=nats://127.0.0.1:4222
  │
  └─→ Passes to run_dynamo_worker.sh:
      ├─ ETCD_ENDPOINTS=http://127.0.0.1:2379
      ├─ NATS_SERVER=nats://127.0.0.1:4222
      ├─ DYN_RUNTIME_JSON_LOGS=1
      ├─ SGLANG_ALLOW_OVERWRITE_LONGER_CONTEXT_LEN=1
      └─ WORKER_IMAGE=local/dynamo-sglang:runtime-json-logs
          ↓
      dynamo-sglang-worker container gets:
          -e DYN_RUNTIME_JSON_LOGS=1
          -e ETCD_ENDPOINTS=http://127.0.0.1:2379
          -e NATS_SERVER=nats://127.0.0.1:4222
          -e SGLANG_ALLOW_OVERWRITE_LONGER_CONTEXT_LEN=1
          --gpus all
```

### Dev Mode (Quick Iteration)

`WORKER_DEV_MODE=1` enables **Python-only hot-reloading** without rebuilding Docker images:

```bash
WORKER_DEV_MODE=1 \
WORKER_DEV_SOURCE_ROOT=/home/jenzhou/Project__AgenticAI/kv_cache_offloading-main/runtime_upstream/dynamo/components/src/dynamo \
./run_dynamo_worker.sh start
```

**What this does**:
- Bind-mounts local Python source into the container:
  - `${WORKER_DEV_SOURCE_ROOT}/common → /workspace/components/src/dynamo/common`
  - `${WORKER_DEV_SOURCE_ROOT}/sglang → /workspace/components/src/dynamo/sglang`
  - `${WORKER_DEV_BINDINGS_ROOT}/runtime → /workspace/lib/bindings/python/src/dynamo/runtime`
- Sets `PYTHONPATH` to prioritize these bind-mounts
- **Use case**: Edit `runtime_logging.py` on the host → restart worker → changes take effect (no rebuild)
- **Limitation**: Only works for Python code changes, not Rust or compiled extensions

### Image Selection Logic

The scripts choose which Docker image to use based on environment variables:

```bash
# Default (smoke test - uses published NVIDIA image)
FRONTEND_IMAGE=${FRONTEND_IMAGE:-nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.0.2}
WORKER_IMAGE=${WORKER_IMAGE:-nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.0.2}

# Instrumented (custom build with blue code)
FRONTEND_IMAGE=local/dynamo-frontend:runtime-json-logs
WORKER_IMAGE=local/dynamo-sglang:runtime-json-logs
```

**When to override**:
- ❌ Smoke test (Step 1): Leave unset → uses published images
- ✅ Instrumented run (Step 3+): Set both → uses custom images with hint logging

---

## Key Files Reference

| Component | File Path | Purpose |
|-----------|-----------|---------|
| **Upload to Hopper** | `upload.sh` | Rsync repo to gracehopper via jump host |
| **Smoke Test** | `run_dynamo_single_host.sh` | Single-host orchestration (head + worker) |
| **Frontend Startup** | `run_dynamo_head.sh` | Start etcd, NATS, Dynamo frontend |
| **Worker Startup** | `run_dynamo_worker.sh` | Start SGLang worker with GPU |
| **Image Builder** | `runtime_instrumentation/build_instrumented_dynamo_images.sh` | Build custom Docker images |
| **Source Prep** | `runtime_instrumentation/prepare_instrumented_dynamo_source.sh` | Fetch + patch upstream Dynamo |
| **Hint Preservation Patch** | `runtime_instrumentation/patches/dynamo_preserve_agent_hints_to_worker.patch` | Blue code layer #1 (Rust) |
| **JSON Logging Patch** | `runtime_instrumentation/patches/dynamo_runtime_json_logging.patch` | Blue code layer #2 (Python) |
| **Benchmark Runner** | `agentbench/deepagents_swebench_single_host.py` | Runs SWE-bench tasks |
| **Hint Construction** | `agentbench/deepagents_app/src/agent.py` | Builds nvext payload |

---

## Common Gotchas & Troubleshooting

### 1. etcd Fails to Start
**Symptom**: Frontend logs `Frontend did not become healthy`

**Diagnosis**:
```bash
docker logs dynamo-etcd
curl -s http://127.0.0.1:2379/health
```

**Fix**: Clean etcd state and restart
```bash
docker rm -f dynamo-etcd
rm -rf ~/kv_cache_offloading/dynamo_head_state/etcd-data
mkdir -p ~/kv_cache_offloading/dynamo_head_state/etcd-data
./run_dynamo_single_host.sh start
```

### 2. Model Never Registers
**Symptom**: `wait_for_model_registration()` times out after 4 minutes

**Diagnosis**:
```bash
# Check if worker is running
docker ps -a | grep dynamo-sglang-worker

# Check worker logs for errors
docker logs dynamo-sglang-worker | tail -100

# Check if frontend can reach etcd
docker logs dynamo-frontend | grep -i etcd
```

**Common Causes**:
- Worker container crashed (GPU incompatibility, OOM)
- Model download failed (missing `HF_TOKEN` or network issue)
- ETCD/NATS endpoints misconfigured (check `ETCD_ENDPOINTS` and `NATS_SERVER`)

### 3. Context Length Exceeded
**Symptom**: `current token count exceeds the model maximum context length of 32768 tokens`

**Fix**: Restart worker with extended context
```bash
./run_dynamo_single_host.sh stop

SGLANG_ALLOW_OVERWRITE_LONGER_CONTEXT_LEN=1 \
DYN_TOOL_CALL_PARSER=hermes \
DYNAMO_MODEL_PATH='Qwen/Qwen2.5-7B-Instruct' \
WORKER_EXTRA_ARGS='--enable-cache-report --enable-priority-scheduling --radix-eviction-policy lru --context-length 65536' \
FRONTEND_IMAGE=local/dynamo-frontend:runtime-json-logs \
WORKER_IMAGE=local/dynamo-sglang:runtime-json-logs \
./run_dynamo_single_host.sh start
```

**Verify**:
```bash
docker inspect dynamo-sglang-worker --format '{{range .Config.Env}}{{println .}}{{end}}' | grep CONTEXT
```

### 4. Architecture Mismatch
**Symptom**: `exec format error` or images won't start on Hopper

**Diagnosis**:
```bash
# Check host architecture
uname -m  # gracehopper is likely aarch64 (ARM)

# Check image architecture
docker image inspect local/dynamo-frontend:runtime-json-logs --format '{{.Architecture}}'
docker image inspect local/dynamo-sglang:runtime-json-logs --format '{{.Architecture}}'
```

**Expected**:
- x86_64 host → `amd64` images
- aarch64 host → `arm64` images

**Fix**: Rebuild images **natively on Hopper** (don't upload pre-built x86 images to ARM machines)

### 5. GPU OOM During Benchmark
**Symptom**: Worker crashes with `CUDA out of memory` during AgentBench run

**Diagnosis**:
```bash
nvidia-smi  # Check GPU memory usage
docker logs dynamo-sglang-worker | grep -i memory
```

**Fixes**:
- **Option A**: Use smaller model (e.g., `Qwen/Qwen2.5-0.5B` instead of `7B`)
- **Option B**: Reduce context length to 32768
- **Option C**: Use task with smaller prompt (higher `--index` in SWE-bench might have shorter tasks)
- **Option D**: Add more GPUs or use larger-memory machine

### 6. Hints Missing from Worker Logs
**Symptom**: AgentBench runs, but `grep hint_probe_id` finds nothing in worker logs

**Diagnosis**:
```bash
# Check if you're using instrumented images
docker inspect dynamo-frontend --format '{{.Config.Image}}'
docker inspect dynamo-sglang-worker --format '{{.Config.Image}}'

# Should output: local/dynamo-frontend:runtime-json-logs, NOT nvcr.io/...

# Check if DYN_RUNTIME_JSON_LOGS was set
docker inspect dynamo-sglang-worker --format '{{range .Config.Env}}{{println .}}{{end}}' | grep DYN_RUNTIME_JSON_LOGS
```

**Fixes**:
- ❌ **Wrong images**: You're using published images (smoke test mode). Rebuild and use `local/dynamo-*:runtime-json-logs`
- ❌ **Missing env var**: Restart with `DYN_RUNTIME_JSON_LOGS=1`
- ❌ **AgentBench not sending hints**: Check `agentbench/deepagents_app/src/agent.py` has `build_phase_hints()` calls

### 7. Docker Build Fails (Disk Full)
**Symptom**: `no space left on device` during `build_instrumented_dynamo_images.sh`

**Diagnosis**:
```bash
df -h /
docker system df
```

**Fix**:
```bash
# Clean up old Docker layers/containers
docker system prune -a

# Use LEAN_FRONTEND=1 to skip benchmark packages
LEAN_FRONTEND=1 DYN_RUNTIME_JSON_LOGS=1 \
./runtime_instrumentation/build_instrumented_dynamo_images.sh
```

### 8. Upload to Hopper Fails
**Symptom**: `upload.sh` hangs or fails with SSH errors

**Diagnosis**:
```bash
# Test jump host connection
ssh -J falcon.7elements.com:1337 ojaiyeob@gracehopper echo "Connected"

# Dry-run to see what would be uploaded
./upload.sh --dry
```

**Common Causes**:
- SSH key not loaded (run `ssh-add ~/.ssh/your_key`)
- Jump host unreachable (check VPN/network)
- Incorrect username/hostname in `upload.sh`

---

## Quick Command Reference

### On Your Local Machine
```bash
# Preview what will be uploaded to Hopper
./upload.sh --dry

# Upload repo to Hopper
./upload.sh
```

### On Hopper Machine

#### Prerequisites Check
```bash
cd ~/kv_cache_offloading
echo "host arch: $(uname -m)"
python3.11 --version
docker --version
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
test -n "${HF_TOKEN:-}" && echo "HF_TOKEN is set" || echo "HF_TOKEN is missing"
```

#### Smoke Test (No Build)
```bash
./run_dynamo_single_host.sh start  # Uses published images
./run_dynamo_single_host.sh test   # Quick validation
curl -fsS http://127.0.0.1:8000/v1/models  # Check model registration
./run_dynamo_single_host.sh stop
```

#### Build Instrumented Images
```bash
./runtime_instrumentation/prepare_instrumented_dynamo_source.sh
LEAN_FRONTEND=1 DYN_RUNTIME_JSON_LOGS=1 ./runtime_instrumentation/build_instrumented_dynamo_images.sh
```

#### Start Instrumented Runtime
```bash
DYN_RUNTIME_JSON_LOGS=1 \
DYN_TOOL_CALL_PARSER=hermes \
DYNAMO_MODEL_PATH='Qwen/Qwen2.5-7B-Instruct' \
DYNAMO_SERVED_MODEL_NAME='Qwen/Qwen2.5-7B-Instruct' \
FRONTEND_IMAGE=local/dynamo-frontend:runtime-json-logs \
WORKER_IMAGE=local/dynamo-sglang:runtime-json-logs \
./run_dynamo_single_host.sh start
```

#### Run First Benchmark
```bash
python3.11 agentbench/deepagents_swebench_single_host.py \
  --app-variant upstream_deploy_coding_agent \
  --frontend-url http://127.0.0.1:8000/v1/chat/completions \
  --model Qwen/Qwen2.5-7B-Instruct \
  --dataset ScaleAI/SWE-bench_Pro \
  --split test \
  --index 0 \
  --prompt-evolution-value-char-limit 1000
```

#### Verify Results
```bash
LATEST_RESULT="$(ls -td agentbench/results/* | head -1)"
grep -R "hint_probe_id\|agent_hints" -n "$LATEST_RESULT" | head -50
cat "$LATEST_RESULT/runtime_hint_alignment_analysis.md"
```

---

## Next Steps After Understanding

1. **Verify prerequisites** on Hopper machine (Step 0)
2. **Run smoke test** to prove basic connectivity (Step 1)
3. **Build instrumented images** to enable hint logging (Step 2)
4. **Start instrumented runtime** with custom images (Step 3)
5. **Run first benchmark** (index 0) to validate end-to-end (Step 4)
6. **Verify results** contain hint_probe_id in worker logs (Step 5)

Once Step 5 succeeds, you've proven the full pipeline works and can run more comprehensive benchmarks.

---

## Summary: The Full Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ LOCAL MACHINE                                                       │
│                                                                     │
│  1. Edit code                                                       │
│  2. ./upload.sh                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ rsync via jump host
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ HOPPER MACHINE (gracehopper)                                        │
│                                                                     │
│  3. Build images:                                                   │
│     ./runtime_instrumentation/prepare_instrumented_dynamo_source.sh │
│     ./runtime_instrumentation/build_instrumented_dynamo_images.sh   │
│                                                                     │
│  4. Start Docker stack:                                             │
│     DYN_RUNTIME_JSON_LOGS=1 \                                       │
│     FRONTEND_IMAGE=local/dynamo-frontend:runtime-json-logs \        │
│     WORKER_IMAGE=local/dynamo-sglang:runtime-json-logs \            │
│     ./run_dynamo_single_host.sh start                               │
│         │                                                           │
│         ├─→ dynamo-etcd container (port 2379)                       │
│         ├─→ dynamo-nats container (port 4222)                       │
│         ├─→ dynamo-frontend container (port 8000)                   │
│         │   └─ [BLUE CODE #1: Rust preprocessor extracts hints]    │
│         └─→ dynamo-sglang-worker container (GPU)                    │
│             └─ [BLUE CODE #2: Python logger emits [RUNTIME_JSON]]  │
│                                                                     │
│  5. Run benchmark:                                                  │
│     python3.11 agentbench/deepagents_swebench_single_host.py \      │
│       --index 0                                                     │
│         │                                                           │
│         └─→ Constructs nvext.agent_hints payload                    │
│             └─→ HTTP POST to :8000/v1/chat/completions              │
│                 └─→ Frontend preprocessor extracts hints            │
│                     └─→ Worker receives hints in extra_args         │
│                         └─→ Worker logs [RUNTIME_JSON] events       │
│                                                                     │
│  6. Verify:                                                         │
│     grep hint_probe_id agentbench/results/*/others/worker_runtime.log│
│     ✅ Success: hints traveled through the stack                    │
└─────────────────────────────────────────────────────────────────────┘
```
