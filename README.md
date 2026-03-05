# Web Agents Research

Research repository for the thesis:
**"Implementation and Evaluation of an Autonomous Web Agent 
in a Controlled Web Application Environment"**

---

## About

This repository contains all research notes, cloned repositories, 
literature, and implementation plans related to autonomous web agents. 
The research covers agent architectures, web perception methods, 
browser infrastructure, and evaluation benchmarks — with the goal 
of implementing and evaluating an autonomous web agent in a controlled 
web application environment using Steel.dev as browser infrastructure.

---

## Repository Structure
```
web-agents-research/
│
├── agents/                    — research notes on individual web agents
├── infrastructure/            — browser infrastructure platforms
├── frameworks/                — automation frameworks
├── benchmarks/                — evaluation benchmarks
├── literature/                — papers, links, and sources
├── notes/                     — personal notes, todo, research questions
│
└── cloned-repositories/       — full cloned GitHub repositories
    ├── browser-use/
    ├── skyvern/
    ├── Agent-E/
    ├── stagehand/
    ├── nanobrowser/
    ├── browserable/
    ├── open-interpreter/
    ├── UI-TARS/
    ├── self-operating-computer/
    ├── OpenAgents/
    ├── WebAgent/
    ├── OmniParser/
    ├── WebVoyager/
    ├── BrowserGym/
    ├── WorkArena/
    ├── WebCanvas/
    ├── webarena/
    ├── steel-browser/
    ├── steel-cookbook/
    ├── awesome-web-agents/
    ├── surf.new/
    └── browser-agents-article/
```

---

## Research Questions

1. Which web perception approach (DOM, Vision, Hybrid) achieves the best results on defined tasks in a controlled environment?
2. How does Steel infrastructure affect the performance and reliability of an autonomous web agent?
3. How successfully does an autonomous web agent execute defined tasks in a controlled web application?

---

## Implementation Plan

Three approaches implemented and compared on identical tasks:

| Approach | Stack |
|----------|-------|
| A | Custom agent — Steel + Playwright + LLM API directly (no framework) |
| B | Browser-Use + Steel |
| C | Steel CLI + steel-browser skill |

---

## Agents Under Research

| Agent | Perception | Open-source |
|-------|-----------|-------------|
| Browser-Use | Hybrid | ✅ |
| Skyvern | Vision | ✅ |
| Agent-E | DOM + distillation | ✅ |
| Stagehand | Hybrid | ✅ |
| OpenAI Operator | Vision | ❌ |
| Magnitude | Vision-first | ❌ |

---

## Key Links

| Resource | Link |
|----------|------|
| Steel Docs | https://docs.steel.dev |
| Steel Leaderboard | https://leaderboard.steel.dev |
| Awesome Web Agents | https://github.com/steel-dev/awesome-web-agents |
| ReAct Paper | https://arxiv.org/abs/2210.03629 |
| WebVoyager Paper | https://arxiv.org/abs/2401.13919 |
| WebArena | https://webarena.dev |
| All Links | /literature/links.md |
