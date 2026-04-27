# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

SplitwiseSim is a discrete-event simulator for LLM serving infrastructure. It models multi-region GPU clusters serving generative LLM workloads with request-level granularity, including scheduling, autoscaling, and long-term resource allocation via MILP.

## Commands

```bash
# Install dependencies (Python 3.11 recommended)
pip install -r requirements.txt

# Run the simulator
python run.py

# Run with Hydra overrides
python run.py trace.filename=ES_26 short_term_scaling=False long_term_scaling=True global_arbiter.arima_traces=$PWD/traces/forecasts/

# Run with automatic timestamped output directory
python run_kunal.py
```

All runtime configuration is driven by `configs/config.yaml` via Hydra. Hydra changes the working directory to the output directory at runtime, so use `get_original_cwd()` (from `hydra.utils`) to resolve relative paths in code.

## Architecture

### Event-driven simulation

`simulator.py` provides the core discrete-event engine. There is a global `sim` singleton. Key API:
- `clock()` — current simulation time
- `schedule_event(delay, action)` — schedule a function to run after `delay` seconds
- `cancel_event(event)` / `reschedule_event(event, delay)`

All component behavior is driven by scheduling/canceling events — there are no threads.

### Request lifecycle

Requests are **DAGs** of `Task` (computation on GPUs) and `Flow` (communication over links) nodes. The `Executor` walks the DAG, submitting nodes as predecessors complete.

**Routing hierarchy:**
```
GlobalRouter → Region → RegionRouter → ModelEndpointRouter → Application → Scheduler → Instance (GPUs)
```

`RequestState` in `request.py` tracks each stage. The same hierarchy is traversed in reverse on completion.

### Key components

| File | Component | Role |
|------|-----------|------|
| `controller.py` | Controller | Collection of Regions; top-level cluster manager |
| `region.py` | Region | Geographic area with a RegionCluster and multiple ModelEndpointRouters |
| `region_cluster.py` | RegionCluster | Physical servers (GPUs) in a region |
| `region_router.py` | RegionRouter | Routes requests to the correct ModelEndpointRouter within a region |
| `model_endpoint_router.py` | ModelEndpointRouter | Per-model router that distributes requests across Applications |
| `application.py` | Application | Logical endpoint serving one model, owns Scheduler + Allocator |
| `scheduler.py` | Scheduler | Schedules Requests onto Instances; spawns Executors. Multiple strategies: JSQ, KV-JSQ, round-robin, mixed-pool, etc. |
| `allocator.py` | Allocator | Autoscales Instances for an Application (currently noop by default) |
| `arbiter.py` | Arbiter | Manages GPU allocation between Applications within a region; supports short-term scaling and spot/preemptible instance logic |
| `global_arbiter.py` | GlobalArbiter | Long-term MILP-based allocation across regions using ARIMA forecast traces |
| `global_router.py` | GlobalRouter | Top-level router; bridges to GlobalArbiter for long-term planning |
| `instance.py` | Instance | A model replica running on specific GPUs; manages task queues, batching, preemption |
| `executor.py` | Executor | Walks a Request's DAG, submitting Tasks/Flows as predecessors complete |
| `long_term_allocation.py` | MilpLongTermAllocation | PuLP-based MILP solver for cross-region GPU allocation |
| `performance_model.py` | PerformanceModel | Estimates Task/iteration duration based on batch size, model, and hardware |
| `model.py` | Model / ModelParallelism | Dataclass capturing model architecture, parallelism config, and memory size |
| `node.py` | Node / NodeState | Base class for DAG nodes (Tasks and Flows) |
| `trace.py` | Trace | Reads request traces from CSV, batches them for memory-efficient streaming into the simulator |
| `initialize.py` | init helpers | Wires up all repos and components from Hydra config |

### Repository pattern

Config loading uses a repository pattern. Each `*_repo.py` file (e.g., `model_repo.py`, `cluster_repo.py`, `hardware_repo.py`) reads YAML configs from a directory and provides lookup functions used during initialization. The `configs/` directory mirrors this structure with subdirectories for each repo type.

### Two-tier scaling

- **Short-term scaling** (`short_term_scaling`): Arbiter adjusts instance counts within a region based on queue pressure and memory utilization. Configured via `scaling_level` (0=off, 1=spot only, 2=inter-model + spot).
- **Long-term scaling** (`long_term_scaling`): GlobalArbiter runs a MILP every `long_term_scaling_interval` seconds, using ARIMA-predicted demand traces to reallocate GPUs across regions. Post-processing strategies: `immediate`, `delay_changes`, `keep_maximum_instances`, `keep_minimum_instances`.

### Key config knobs in `config.yaml`

- `feed_async` / `feed_async_granularity`: inject async/opportunistic requests when memory utilization is low
- `scaling_level`: 0/1/2 for autoscaling aggressiveness
- `scaling_interval`: minimum seconds between scaling events (-1 to disable)
- `siloed`: if True, appends workload type suffix to model names (for prod/dev separation)

## Output files

Simulation writes results to the Hydra output directory (see `output_dir` in config.yaml):
- `summary.csv` — per-application aggregate metrics
- `detailed/{application_id}.csv` — per-request metrics
- `request_nodes.csv` — per-node (task/flow) level metrics
- `instances/` — instance-level logs (with `debug: True`)
- `global_ariber_logs.csv` — per-region, per-model GPU allocation over time
