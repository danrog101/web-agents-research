# WebArena Benchmark Report

## Executive Summary

WebArena is a **self-hosted, Docker-based web benchmark** providing realistic, interactive replicas of real websites for evaluating autonomous agents. Introduced by Zhou et al. (CMU, 2023, ICLR 2024), it is the **most rigorous controllable web benchmark** because: (1) agents actually execute actions on live websites (not snapshots), (2) evaluations check functional correctness (did the task succeed?), not action string matching, and (3) environments reset to a clean state after each run. As of 2025, the highest documented score is ~61.7% (IBM CUGA), compared to human performance of 78.24%.

---

## 1. Background & Motivation

### Why WebArena Over Mind2Web?

Mind2Web's limitation: it's a static dataset of pre-captured HTML snapshots. Agents are evaluated on whether their action predictions match the crowdsourced annotation — not whether the task was actually completed.

WebArena's innovation: **functional correctness evaluation**. The agent must actually perform the task in a live (containerised) environment, and an automated evaluator checks whether the goal was achieved — regardless of what action sequence was used.

### Paper

> Zhou, S., Xu, F.F., Zhu, H., Zhou, X., Lo, R., Sridhar, A., Cheng, X., Bisk, Y., Fried, D., Alon, U., & others.  
> **WebArena: A Realistic Web Environment for Building Autonomous Agents**  
> ICLR 2024  
> arXiv:2307.13854

---

## 2. Dataset Structure

### 812 Tasks Across 6 Environments

```
WebArena Environments (Docker containers):
├─ Shopping (OneStopShop)
│   Port 7770 — e-commerce, product search, cart management
├─ Shopping Admin (WooCommerce CMS)
│   Port 7780/admin — order management, reports, customer data
├─ Reddit (Postmill)
│   Port 9999 — forums, posting, voting, moderation
├─ GitLab (GitLab CE)
│   Port 8023 — repositories, issues, merge requests, CI
├─ Wikipedia (Kiwix offline)
│   Port 8888 — encyclopaedia, article lookup
├─ Map (OpenStreetMap-based)
│   Port 3000 — navigation, location lookup
└─ Homepage (Portal)
    Port 4399 — links to all other environments
```

**Task breakdown:**

| Domain | Tasks | Examples |
|--------|-------|---------|
| Shopping | ~160 | Find cheapest product with feature X; compare prices |
| Shopping Admin | ~160 | Report monthly orders Jan-May 2023 |
| Reddit | ~140 | Find most upvoted post in r/cooking this week |
| GitLab | ~180 | Create issue + assign to user; list open PRs |
| Cross-site | ~170 | Find GitLab project URL then post it to Reddit |

### Task Format

```json
{
    "task_id": 108,
    "sites": ["shopping_admin"],
    "task_type": "RETRIEVE",
    "intent": "Get the monthly count of successful orders 01/2023-05/2023",
    "start_url": "http://localhost:7780/admin",
    "eval": {
        "eval_types": ["string_match"],
        "reference_answers": {
            "exact_match": "Jan: 12, Feb: 7, March: 5, April: 9, May: 5"
        }
    }
}
```

### Task Types

| Type | Description | Eval method |
|------|-------------|-------------|
| `RETRIEVE` | Extract information | String/URL match |
| `SITE_NAVIGATION` | Navigate to specific page | URL match |
| `FORM_MANIPULATION` | Fill form, submit data | Database check |
| `MULTI_SITE` | Cross-application tasks | Combined checks |

---

## 3. Environment Setup

```bash
# Prerequisites: Docker, 16GB+ RAM recommended

# Option 1: Docker Compose (all environments)
git clone https://github.com/web-arena-x/webarena
cd webarena

# Launch all Docker containers
docker compose up -d

# Set environment URLs
export SHOPPING="localhost:7770"
export SHOPPING_ADMIN="localhost:7780"
export REDDIT="localhost:9999"
export GITLAB="localhost:8023"
export WIKIPEDIA="localhost:8888/wikipedia_en_all_maxi_2022-05/A/User:The_other_Kiwix_guy/Landing"
export MAP="localhost:3000"
export HOMEPAGE="localhost:4399"

# Option 2: AWS AMI (pre-installed)
# Amazon Machine Image: ami-XXXXXXXX (see repo README)
# Spins up EC2 with all environments pre-configured

# Option 3: WebArena-Verified (recommended for new work)
pip install webarena-verified
webarena-verified env start --site shopping
```

### Gym Interface

```python
from browser_env import ScriptBrowserEnv, create_id_based_action

# Init environment
env = ScriptBrowserEnv(
    headless=False,
    observation_type="accessibility_tree",  # or "html", "image"
    current_viewport_only=True,
    viewport_size={"width": 1280, "height": 720},
)

# Load task configuration
config_file = "config_files/108.json"
obs, info = env.reset(options={"config_file": config_file})

# Agent loop
action = create_id_based_action("click [15]")  # click element id=15
obs, reward, terminated, truncated, info = env.step(action)
# reward: 1.0 = task completed, 0.0 = not completed
```

### Action Format

```python
# WebArena action types
"CLICK [id]"          # Click element by ID from accessibility tree
"TYPE [id] [text]"    # Type text into element
"SCROLL [id] [dir]"   # Scroll element (up/down)
"PRESS [key]"         # Key press (Enter, Tab, etc.)
"GOTO [url]"          # Navigate to URL
"STOP [answer]"       # Task complete, provide answer
```

---

## 4. Evaluation System

### Functional Correctness (not action matching)

```python
# evaluate.py — evaluates whether the GOAL was achieved

def evaluate_task(agent_answer: str, task: Task, env_state) -> float:
    """
    Returns 1.0 (success) or 0.0 (failure).
    Does NOT compare action sequences — only checks if goal achieved.
    """
    if task.task_type == "RETRIEVE":
        return string_match(agent_answer, task.reference_answer)
    elif task.task_type == "FORM_MANIPULATION":
        return check_database_state(task.expected_db_state, env_state)
    elif task.task_type == "SITE_NAVIGATION":
        return url_match(env_state.current_url, task.expected_url)
```

### Reset Mechanism

```bash
# Reset environment to initial state (critical for reproducibility)
docker stop webarena-shopping && docker start webarena-shopping
docker stop webarena-gitlab  && docker start webarena-gitlab
# All data is restored from Docker image
```

---

## 5. WebArena-Verified (ServiceNow, 2025)

An improved, re-audited version of WebArena released by ServiceNow:

```bash
pip install webarena-verified
# or
docker pull ghcr.io/servicenow/webarena-verified:latest
```

**Improvements over original:**
- Every task, reference answer, and evaluator manually reviewed and corrected
- Deterministic JSON-based scoring (no LLM-as-judge)
- Network events-based evaluation (offline replay possible)
- 92% smaller Docker images
- **WebArena-Verified Hard** subset: 258 tasks, difficulty-prioritised

```python
# WebArena-Verified evaluation
from webarena_verified import evaluate_task_from_log

result = evaluate_task_from_log(
    task_id=108,
    agent_log_dir="./agent_logs/108/",
)
print(result.status)         # "SUCCESS" or error code
print(result.score)          # 1.0 or 0.0
print(result.evaluator_details)
```

---

## 6. SOTA Performance History

| Date | Agent/System | Score |
|------|-------------|-------|
| Jul 2023 (paper) | GPT-4 + CoT | 11.7% |
| Jul 2023 | GPT-3.5 + CoT | 8.75% |
| 2024 | Tree-search agents | ~25-35% |
| 2024 | GPT-4o + retrieval | ~36.7% |
| Feb 2025 | IBM CUGA | **61.7%** |
| — | Human | 78.24% |

The jump from ~11% to ~61% in 2 years reflects the impact of GPT-4V, better prompting, and RL-trained models.

---

## 7. Architecture Diagram

```
WebArena
  ├─ 6 Docker containers (self-hosted websites)
  │   ├─ OneStopShop (port 7770)
  │   ├─ WooCommerce Admin (7780)
  │   ├─ Postmill/Reddit (9999)
  │   ├─ GitLab CE (8023)
  │   ├─ Wikipedia/Kiwix (8888)
  │   └─ OpenStreetMap (3000)
  ├─ 812 tasks (config_files/*.json)
  ├─ ScriptBrowserEnv (Gym interface)
  │   ├─ Playwright-powered browser
  │   ├─ Accessibility tree extraction
  │   └─ Action execution + state capture
  └─ Evaluation Harness
      ├─ String match evaluator
      ├─ URL match evaluator
      ├─ Database state evaluator
      └─ Functional correctness (not action matching)
```

---

## 8. Key Technical Decisions

**Why Docker for environments?**
Docker images are fully reproducible — same website, same data, same initial state every run. Any agent can reset to the exact same environment for fair comparison.

**Why functional evaluation (not action matching)?**
Action matching is brittle — there are many valid paths to the same goal. Checking whether the database changed, whether the correct page was reached, or whether the correct answer was extracted is more robust and more representative of real-world success.

**Why cross-site tasks?**
Real-world work often spans multiple applications (look up a GitLab repo URL, then post it to Reddit). Cross-site tasks test this higher-order capability.

---

## 9. Limitations & Known Issues

- **Heavy setup** — 6 Docker containers, 16GB+ RAM, 30+ minutes to initialise
- **Environment drift** — GitLab updates may change UI structure
- **Time-sensitive data** — Wikipedia snapshot from 2022, some data is stale
- **Limited domains** — 6 types vs 31 in Mind2Web
- **Reset overhead** — each evaluation run requires Docker restart for clean state

**License:** MIT (code), research use (data)  
**Repository:** https://github.com/web-arena-x/webarena  
**WebArena-Verified:** https://github.com/ServiceNow/webarena-verified  
**Snapshot date:** 2026-03-18
