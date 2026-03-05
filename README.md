# Web Agents Research

Research repository for the thesis:
**"Implementation and Evaluation of an Autonomous Web Agent 
in a Controlled Web Application Environment"**

University thesis | 2025/2026

---

## About

This repository contains all research notes, literature, 
cloned repositories, and implementation plans for my thesis. 
The research focuses on autonomous web agents — their 
architectures, perception methods, and evaluation in a 
controlled environment using Steel.dev as browser 
infrastructure.

---

## Research Questions

1. Which web perception approach (DOM, Vision, Hybrid) 
   achieves the best results on defined tasks?

2. How does Steel infrastructure affect the performance 
   and reliability of an autonomous web agent?

3. How successfully does an autonomous web agent execute 
   defined tasks in a controlled web application?

---

## Implementation Plan

Three approaches implemented and compared on identical tasks:

| Approach | Stack | Description |
|----------|-------|-------------|
| A | Steel + Playwright + LLM API | Custom agent, no framework |
| B | Browser-Use + Steel | Framework-based agent |
| C | Steel CLI + steel-browser skill | Steel native solution |

---

## Agents Under Research

### Selected for detailed comparison (6)

| Agent | Perception | Open-source | Best for |
|-------|-----------|-------------|----------|
| Browser-Use | Hybrid | ✅ | Development, research |
| Skyvern | Vision | ✅ | Production, login flows |
| Agent-E | DOM + distillation | ✅ | Clean web apps |
| OpenAI Operator | Vision (CUA) | ❌ | Non-technical users |
| Magnitude | Vision-first | ❌ | Critical accuracy tasks |
| Stagehand | Hybrid | ✅ | Quick integration |

### Full inventory
See `/agents` folder for all researched agents.

---

## Repository Structure
```
web-agents-research/
│
├── README.md
│
├── agents/               ← research notes per agent
├── infrastructure/       ← browser infrastructure platforms  
├── frameworks/           ← automation frameworks
├── benchmarks/           ← evaluation benchmarks
├── literature/           ← papers, links, sources
├── notes/                ← personal notes, todo, ideas
│
└── cloned-repositories/  ← full cloned GitHub repos
    ├── browser-use/
    ├── skyvern/
    ├── agent-e/
    ├── stagehand/
    ├── steel-browser/
    └── ...
```

---

## Key Resources

| Resource | Link |
|----------|------|
| Steel Docs | https://docs.steel.dev |
| Steel Leaderboard | https://leaderboard.steel.dev |
| Awesome Web Agents | https://github.com/steel-dev/awesome-web-agents |
| Field Guide | https://github.com/danrog101/browser-agents-article |
| ReAct Paper | https://arxiv.org/abs/2210.03629 |
| WebVoyager Paper | https://arxiv.org/abs/2401.13919 |
| WebArena | https://webarena.dev |
| All Links | /literature/links.md |

---

## Progress

- [x] Repository setup
- [ ] Agent research notes complete
- [ ] Literature collected
- [ ] Theoretical chapter written
- [ ] Implementation A — custom agent
- [ ] Implementation B — Browser-Use + Steel
- [ ] Implementation C — Steel CLI
- [ ] Evaluation complete
- [ ] Thesis written
