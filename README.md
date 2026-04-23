---
title: AI Incident OpenEnv
emoji: 🤖
colorFrom: blue
colorTo: green
sdk: docker
pinned: false
tags:
- openenv
---

# AI Incident Response — OpenEnv Environment

A cmpetitive OpenEnv submission for the Meta × Hugging Face hackathon. This environment simulates **distributed system incident response** — agents must reason over cascading failures, misleading logs, and dependency chains to restore a multi-service system. It is specifically designed to defeat pattern-matching and reward genuine causal reasoning.

---

## What Makes This Novel

Most existing environments test "find the broken thing and fix it." This environment tests **why** something broke, not just **what** broke.

Three design principles differentiate it:

1. **Misleading signal over correct signal.** In `hard-bad-deployment`, the logs produce 8 lines implicating the database for every 1 line implicating auth. A correct agent must suppress the dominant false signal and reason from weaker evidence.

2. **Dynamic environment that degrades without agent action.** Every 2–3 steps on hard tasks, a healthy service autonomously degrades due to the unresolved root cause spreading. The agent races against a deteriorating system, not a static puzzle. Waiting to observe is punished by the environment itself.

3. **Diagnosis gates and honeypot actions.** On `hard-bad-deployment`, running `optimize_db` on the database — the intuitively "correct" action given the logs — actively worsens the system (increases auth error rate, raises system strain). A successful rollback of auth requires prior diagnosis (`check_logs auth` + `check_metrics auth`). Blind fixes produce only partial recovery.

---

## Environment Details

### Service Topology

```
db ──► auth ──► frontend
              ↗
payments ────
```

| Service | Role | Task-specific dependencies |
|:---|:---|:---|
| auth | Authentication gateway | db (cascading-failure), payments (ambiguous) |
| payments | Transaction processing | shared DB pool (ambiguous) |
| db | Persistence layer | root cause in cascading-failure |
| frontend | User interface | auth + payments |

### Observation Space

| Field | Type | Description |
|:---|:---|:---|
| `services` | list | Per-service state: `name`, `status`, `latency`, `error_rate` |
| `logs` | list[str] | Last 10 log lines — may contain misleading entries |
| `alerts` | list[str] | Active alerts (CRITICAL / WARNING / INFO) |
| `system_health` | float 0–1 | Fraction of services currently UP |
| `system_stability` | float 0–1 | Composite stability score (service states + strain + alerts) |
| `system_strain` | float 0–1 | Infrastructure stress from bad actions and unresolved failures |
| `dependencies` | dict | Live dependency graph for the current task |
| `time_step` | int | Current step in the episode |

### Action Space

| Action | Target | Effect |
|:---|:---|:---|
| `restart_service` | service name | Restarts a DOWN service. Does NOT fix DEGRADED state. |
| `rollback_service` | service name | Rolls back to last stable version. Required fix for bad deployments. On `hard-bad-deployment`, requires prior diagnosis to fully succeed. |
| `scale` | service name | Scales a DEGRADED service up. Costs 3.0. |
| `optimize_db` | `db` | Fixes a degraded DB. **Honeypot on `hard-bad-deployment`** — worsens state if DB is healthy. |
| `check_logs` | service name | Adds target to `diagnosed_targets`. Returns updated log tail. |
| `check_metrics` | service name | Returns latency + error_rate per service. Required for diagnosis gate on `hard-bad-deployment`. |
| `isolate_service` | service name | Removes service from dependency graph temporarily. Increases latency on dependents. |
| `drain_traffic` | service name | Redirects traffic away from a degraded service. Reduces frontend error rate. |
| `restore_traffic` | service name | Reverses drain. |
| `escalate` | — | Ends episode immediately. Returns partial score proportional to system health. |
| `ignore` | — | Takes no action. System continues to degrade. |

### Reward Structure

Step-level rewards are calculated per action outcome:

| Outcome | Reward |
|:---|:---|
| Correct fix (root cause) | +0.40 to +0.75 |
| Temporary / partial fix | +0.05 to +0.20 |
| Diagnosis (check_logs / check_metrics) | +0.05 to +0.10 |
| Wrong fix | −0.20 to −0.40 |
| Useless action | −0.20 |
| All services UP | +0.30 bonus |
| Stability improvement | +0.20 bonus |

**Episode final score formula:**

```
final_score = 0.40 × success
            + 0.15 × efficiency
            + 0.20 × cost_efficiency
            + 0.05 × stability
            + 0.20 × sequence
```

The **sequence score** is the most discriminating component. It rewards early root cause identification, correct fix ordering, and penalises:
- Fixing symptoms before the root cause
- Spamming diagnosis actions without fixing anything (≥3 consecutive → hard cap at 0.35 final score)
- Rolling back without prior diagnosis on gated tasks

### Anti-Reward-Hacking Mechanisms

Three explicit protections against gaming the reward signal:

1. **Observation loop penalty.** After 3 consecutive `check_logs`/`check_metrics` with no fix attempt, system strain increases (+0.25), stability drops (−0.15), and a warning log is emitted. Final score is hard-capped at 0.35 for agents that never attempt a fix.

2. **Diagnosis gate.** On `hard-bad-deployment`, `rollback_service auth` without prior `check_logs auth` + `check_metrics auth` produces only a `partial_fix` (reward +0.05, service remains DEGRADED). The `root_cause_fixed` success condition awards only 0.5× credit for blind rollbacks.

3. **Honeypot action.** `optimize_db` on a healthy database (as in `hard-bad-deployment`) is scored as `wrong_fix`, increases auth error rate by +0.10, and raises system strain by +0.30. It counts toward the bad action limit.

---

## Tasks

### easy-auth-down
**Difficulty:** Easy | **Max steps:** 5

Auth service is DOWN. Restart it.

- Initial state: `auth=DOWN`, `payments=UP`, `frontend=UP`
- Success: all services UP, error_rate < 0.05
- Failure: >2 bad actions, steps exceeded
- Correct path: `check_logs auth` → `restart_service auth`

---

### medium-payments-degraded
**Difficulty:** Medium | **Max steps:** 8

Payments is DEGRADED under load. Scale it without over-spending.

- Initial state: `auth=UP`, `payments=DEGRADED`, `frontend=UP`
- Success: no service DOWN, latency < 200ms, total cost < 15
- Failure: >3 bad actions, stability < 0.4
- Correct path: `check_metrics payments` → `scale payments`

---

### hard-bad-deployment ⚠️
**Difficulty:** Hard | **Max steps:** 10

`auth-v2.1.0` introduced a goroutine leak. 8 of 12 initial log lines blame the database. The DB is perfectly healthy. Running `optimize_db` makes things worse.

- Initial state: `auth=DEGRADED (error_rate=0.9)`, `db=UP`, `payments=UP`, `frontend=UP`
- Success: all services UP, root cause fixed (with diagnosis gate)
- Failure: >3 bad actions, stability < 0.3
- Correct path: `check_logs auth` → `check_metrics auth` → `check_logs db` → `rollback_service auth`
- Trap path: `optimize_db db` → **wrong_fix**, strain +0.30, auth error rate worsens

---

### hard-cascading-failure
**Difficulty:** Hard | **Max steps:** 10

DB degradation is causing auth to go DOWN and payments to degrade. Misleading logs blame payments and auth throughout the episode. Environment autonomously degrades healthy services every 2 steps if root cause is unresolved.

- Initial state: `db=DEGRADED`, `auth=DOWN`, `payments=DEGRADED`, `frontend=UP`
- Success: stability ≥ 0.9, root cause (db) fixed, strain < 0.3
- Failure: >3 bad actions, stability < 0.3
- Correct path: `check_logs db` → `optimize_db db` → wait for cascading recovery

---

### hard-cascading-ambiguous
**Difficulty:** Hard | **Max steps:** 15

Both auth and payments are DEGRADED, frontend is DOWN. A `payments-v3.4.1` deployment broke a shared connection pool. Fixing auth first causes it to **re-degrade within 2 steps**.

- Initial state: `auth=DEGRADED`, `payments=DEGRADED`, `db=UP`, `frontend=DOWN`
- Success: all services UP, error_rate < 0.1
- Failure: >4 bad actions, stability < 0.25
- Correct path: `check_logs payments` → `check_metrics payments` → `rollback_service payments`
- Wrong path: `restart_service auth` → auth recovers → re-degrades at step +2

---

### hard-latent-root-cause
**Difficulty:** Hard | **Max steps:** 12

Auth is visibly DEGRADED. Payments is UP. Every surface log blames auth. The true root cause is a latent issue in the payments persistence layer that is not directly visible. Fixing auth produces a fake recovery that degrades again in 2 steps.

- Initial state: `auth=DEGRADED (error_rate=0.5)`, `payments=UP`, `frontend=UP`
- True root cause: `payments`
- Surface symptom: `auth`
- Success: all services UP, root cause fixed
- Failure: >4 bad actions, stability < 0.25
- Correct path: `check_logs payments` → `check_metrics payments` → `restart_service payments` or `rollback_service payments`

---

## Example Trajectory (hard-bad-deployment)

```
Step 1  check_logs auth       → diagnosed_targets: {auth}
        Logs reveal goroutine leak at 8,412 goroutines (expected <500)

Step 2  check_metrics auth    → diagnosis gate condition 1 satisfied
        Metrics: auth error_rate=0.91, latency=1100ms

Step 3  check_logs db         → diagnosed_targets: {auth, db}
        Logs: DB active_connections=42/200, replication_lag=2ms — healthy

Step 4  rollback_service auth → diagnosis gate passed → correct_fix
        auth recovers: status=UP, error_rate=0.01
        Episode ends. Final score: ~0.82
```

**Trap trajectory (same task):**

```
Step 1  optimize_db db        → wrong_fix (DB is healthy)
        auth error_rate: 0.90 → 1.00, system_strain +0.30

Step 2  restart_service auth  → partial_fix (rollback required)
        auth remains DEGRADED

Step 3  rollback_service auth → partial_fix (diagnosis gate not passed)
        auth remains DEGRADED

Step 4  bad_actions=3 → episode terminates, final score capped at 0.30
```

---

## Baseline Scores

| Task | Baseline Score | Model | Notes |
|:---|:---|:---|:---|
| easy-auth-down | 0.82 | llama-3.1-8b-instant | Solves reliably |
| medium-payments-degraded | 0.71 | llama-3.1-8b-instant | Occasional over-scaling |
| hard-bad-deployment | 0.41 | llama-3.1-8b-instant | Frequently hits honeypot |
| hard-cascading-ambiguous | 0.31 | llama-3.1-8b-instant | Re-degradation confuses agent |
| hard-latent-root-cause | 0.28 | llama-3.1-8b-instant | Surface symptom dominates |

*Reproduce with:* `python3 inference.py`

---

## Failure Type Classification

The grader classifies each episode into one of the following types, visible in the score breakdown:

| Type | Condition |
|:---|:---|
| **Efficient Reasoner** | Root cause fixed within 2 steps, no symptom fixes, no observation loop |
| **Symptom Chaser** | Fixed downstream services before root cause, triggered re-degradation |
| **Lucky Guesser** | Fixed root cause without prior diagnosis |
| **Stuck in Observation Loop** | ≥3 consecutive diagnosis actions with no fix attempt |
| **Late Corrector** | Root cause eventually fixed, but after step 5 |

---

## Setup

### Local

```bash
git clone https://github.com/roonakyadav/meta-hackathon
cd meta-hackathon
pip install -r requirements.txt
python3 -m api.main
```

### Docker

```bash
docker build -t ai-incident-openenv .
docker run -p 7860:7860 ai-incident-openenv
```

### Inference (Hugging Face Space)

```bash
API_BASE_URL=https://roonakyadav-ai-incident-openenv-trial.hf.space \
MODEL_NAME=llama-3.1-8b-instant \
HF_TOKEN=your_key \
python3 inference.py
```

---

## API Reference

| Endpoint | Method | Body | Description |
|:---|:---|:---|:---|
| `/reset` | POST | `{"task_id": "...", "seed": 42}` | Start a new episode |
| `/step` | POST | `{"action_type": "...", "target": "..."}` | Take one action |
| `/state` | GET | — | Read current state (no side effects) |
| `/tasks` | GET | — | List all available tasks |

**Live environment:** https://roonakyadav-ai-incident-openenv-trial.hf.space

**GitHub:** https://github.com/roonakyadav/meta-hackathon