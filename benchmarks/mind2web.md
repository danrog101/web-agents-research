
# Mind2Web Benchmark Report

## Executive Summary

Mind2Web is the **first dataset for generalist web agents** — 2,350 tasks spanning 137 real websites across 31 domains, introduced by Ohio State University's NLP group (Deng et al., NeurIPS 2023, Spotlight). Unlike prior benchmarks that used toy simulated websites, Mind2Web collected tasks from real, live websites and crowdsourced step-by-step action sequences. It also introduced **MindAct** — a two-stage LLM pipeline (small model for HTML element filtering, large model for action selection) that established the baseline methodology for DOM-based web agents.

---

## 1. Background & Motivation

### The Problem Mind2Web Solved

Before Mind2Web, web agent benchmarks had two critical limitations:

1. **Simulated websites only** (MiniWoB, WebArena) — agents that learned on toy environments didn't transfer to real websites
2. **Narrow task distributions** — fewer websites, fewer domains, less diversity

Mind2Web addressed both: real websites, massive diversity, crowdsourced annotations.

### Paper

> Deng, X., Gu, Y., Zheng, B., Chen, S., Stevens, S., Wang, B., Sun, H., & Su, Y.  
> **Mind2Web: Towards a Generalist Agent for the Web**  
> NeurIPS 2023, Datasets and Benchmarks Track (Spotlight)  
> arXiv:2306.06070

---

## 2. Dataset Structure

### Scale and Coverage

```
Total tasks:    2,350 open-ended tasks
Websites:       137 real-world websites
Domains:        31 (travel, shopping, finance, health, entertainment, ...)
Action types:   Click, Type, Select
Average steps:  ~4-8 per task (range: 1-20+)
```

### Task Format

Each instance contains:

```json
{
    "task_id": "travel_001",
    "website": "https://www.booking.com",
    "domain": "travel",
    "task": "Book a roundtrip on July 1 from Mumbai to London and vice versa on July 5 for two adults and a 12-year-old in premium economy. If a flight is not available, search through the calendar for available flights.",
    "action_sequence": [
        {
            "action_uid": "a1b2c3d4",
            "operation": {
                "op": "CLICK",
                "original_op": "CLICK",
                "value": ""
            },
            "pos_candidates": ["element_html_1", "element_html_2"],
            "neg_candidates": ["element_html_3", "..."]
        }
    ]
}
```

**Action operations:**
- `CLICK` — click an element (also covers HOVER)
- `TYPE` — enter text into a field
- `SELECT` — choose a dropdown option

### Three Generalisation Splits

Mind2Web evaluates three levels of generalisation:

| Split | Description | Test Set Size |
|-------|-------------|---------------|
| **Cross-Task** | Same website, different task | ~500 tasks |
| **Cross-Website** | Same domain, different website | ~500 tasks |
| **Cross-Domain** | Different domain entirely | ~500 tasks |

This hierarchy tests whether agents generalise from memorisation (cross-task) to true zero-shot transfer (cross-domain).

---

## 3. MindAct — Baseline Agent Architecture

### Two-Stage Pipeline

Raw HTML of real websites can contain 10,000+ elements — too many to fit in any LLM's context. MindAct solves this with a two-stage approach:

```
Stage 1: Small LM Filtering (Flan-T5)
    Input:  Full HTML page + task description
    Output: Top-K candidate elements (ranked by relevance)
    Method: Fine-tuned DeBERTa-v3-base
    
Stage 2: LLM Action Selection (GPT-3.5 / GPT-4)
    Input:  Task + filtered elements (K candidates)
    Output: Selected element + action type + value
    Method: Multi-choice QA prompt
```

```python
# Simplified MindAct pipeline
def mindact_step(task: str, html: str, llm) -> Action:
    # Stage 1: Filter elements with small LM
    all_elements = parse_html_elements(html)        # ~1000+ elements
    candidates = small_lm_rank(task, all_elements)  # → top 20 candidates
    
    # Stage 2: LLM selects from candidates
    prompt = build_mcqa_prompt(task, candidates)
    response = llm.complete(prompt)
    return parse_action(response)
```

### MindAct Performance (Original Paper)

| Model | Cross-Task SR | Cross-Website SR | Cross-Domain SR |
|-------|-------------|-----------------|-----------------|
| GPT-3.5 (zero-shot) | 14.4% | 11.5% | 10.8% |
| GPT-3.5 + small LM | 32.0% | 24.6% | 23.5% |
| GPT-4 + small LM | 38.7% | 30.5% | 28.5% |
| Human upper bound | ~90% | ~85% | ~80% |

---

## 4. Dataset Repository Structure

```
Mind2Web/
├── data/
│   ├── train/
│   │   └── train.json              # Training set (HuggingFace hosted)
│   └── test/
│       ├── test_task.json          # Cross-task split
│       ├── test_website.json       # Cross-website split
│       └── test_domain.json        # Cross-domain split
│       # test splits are zip-encrypted: password = "mind2web"
├── src/
│   ├── data_utils/
│   │   └── dom_utils.py            # HTML parsing utilities
│   ├── evaluate.py                 # Evaluation script
│   └── model/
│       └── mindact.py              # MindAct implementation
└── README.md
```

---

## 5. Evaluation Metrics

```python
# Three metrics (all element-level)

# 1. Element Accuracy
# Fraction of steps where the agent selected the correct element
element_acc = correct_elements / total_steps

# 2. Operation F1
# F1 score between predicted and ground truth operations (CLICK/TYPE/SELECT)
operation_f1 = f1_score(predicted_ops, true_ops)

# 3. Step Success Rate (SR)
# Fraction of steps where BOTH element AND operation are correct
step_sr = correct_steps / total_steps

# Macro vs Micro:
# Macro: average over tasks (recommended — avoids bias toward long tasks)
# Micro: average over steps
# NOTE: Always use MACRO for paper comparison
```

---

## 6. Extensions and Follow-ups

### Multimodal-Mind2Web (2024)

```
osunlp/Multimodal-Mind2Web (HuggingFace)
- Original HTML documents paired with corresponding screenshots
- Enables evaluation of both text-only and vision-based agents
- Removes need to download the full raw dump
```

### Online-Mind2Web (2025)

```
OSU-NLP-Group/Online-Mind2Web (COLM 2025)
- 300 tasks from 136 popular live websites (updated version)
- Actually executes agent actions on live websites
- Evaluated with WebJudge (o4-mini) — 86% agreement with human
- Addresses the "illusion of progress" problem:
  agents that score well on static Mind2Web
  may fail when websites actually respond to actions
- Includes WebJudge-7B reward model for RL training
```

### Mind2Web-2 (NeurIPS 2025)

```
OSU-NLP-Group/Mind2Web-2
- Focus on "agentic search" — tasks requiring multi-step web research
- Novel Agent-as-a-Judge evaluation framework
- Long-horizon tasks: synthesise information from multiple sources
- Citation-backed answers required
```

---

## 7. Architecture Diagram

```
Mind2Web Dataset
  ├─ 2,350 tasks from 137 websites
  ├─ 31 domains
  └─ Three splits:
      ├─ Cross-Task  (same site, new task)
      ├─ Cross-Website (same domain, new site)
      └─ Cross-Domain (new domain entirely)
          ↓
MindAct Evaluation Pipeline
  ├─ Stage 1: DeBERTa element ranker
  │   Input:  Full HTML → Top-K candidates
  └─ Stage 2: GPT-3.5/4 action selector
      Input:  K candidates + task
      Output: Selected element + operation
          ↓
Metrics:
  ├─ Element Accuracy
  ├─ Operation F1
  └─ Step Success Rate (MACRO recommended)
```

---

## 8. Significance in the Field

Mind2Web catalysed the web agent research community by providing:
1. **The first real-website training/evaluation dataset** (137 sites vs 10-20 in prior work)
2. **The two-stage filtering approach** that became standard for DOM-based agents
3. **Three-way generalisation hierarchy** that became the standard evaluation protocol
4. **A released dataset** enabling reproducible research

**Limitations acknowledged in paper:**
- Static dataset — agent sees pre-captured HTML snapshots, not live websites (addressed by Online-Mind2Web)
- Crowdsourced actions may not be optimal paths
- No measurement of whether the task was actually completed — only whether each step matched the annotation

**Paper:** NeurIPS 2023 Spotlight. arXiv:2306.06070  
**License:** CC BY 4.0 (dataset), MIT (code)  
**Repository:** https://github.com/OSU-NLP-Group/Mind2Web  
**HuggingFace:** osunlp/Mind2Web  
**Snapshot date:** 2026-03-18  

*Generated: 2026-03-18. Based on: GitHub README, NeurIPS 2023 paper, Online-Mind2Web repo, and Mind2Web-2 repo.*
