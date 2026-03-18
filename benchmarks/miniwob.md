
# MiniWoB++ Benchmark Report

## Executive Summary

MiniWoB++ (Mini World of Bits Plus Plus) is the **foundational benchmark for web agent research**, introduced by Shi et al. (OpenAI, 2017) and extended by Liu et al. (2018). It consists of **104 simulated web tasks** implemented as self-contained HTML pages — form filling, button clicking, calendar navigation, text editing, and more. Despite its toy nature, MiniWoB++ established the paradigm of reinforcement learning for web automation and served as the primary benchmark for years. It remains relevant today as a **fast, cheap, fully reproducible** evaluation environment for testing basic web interaction capabilities.

---

## 1. Background & Motivation

### The Original "World of Bits" (2017)

Shi et al. (OpenAI) introduced WoB in 2017 — the first attempt to treat web interaction as an RL problem. The core insight: the web is a universal interface, and agents that can use the web can do almost anything. WoB created 80 microtask environments (HTML+JS), each with a reward signal.

**Paper:**
> Shi, T., Karpathy, A., Fan, L., Hernandez, J., & Liang, P.  
> **World of Bits: An Open-Domain Platform for Web-Based Agents**  
> ICML 2017

### MiniWoB++ Extension (2018)

Liu et al. extended WoB to 104 tasks with:
- Natural language task specifications
- More complex multi-step tasks
- Improved evaluation with string matching

**Paper:**
> Liu, E., Guu, K., Pasupat, P., Shi, T., & Liang, P.  
> **Reinforcement Learning on Web Interfaces Using Workflow-Guided Exploration**  
> ICLR 2018

---

## 2. Dataset Structure

### 104 Tasks by Category

```
MiniWoB++/
├── html/miniwob/           # Task HTML implementations
│   ├── click-button.html
│   ├── click-dialog.html
│   ├── click-menu.html
│   ├── login-user.html
│   ├── book-flight.html    # Complex multi-step
│   ├── enter-date.html
│   ├── search-engine.html
│   └── ... (104 total)
└── python/computergym/miniwob/
    └── miniwob_env.py      # OpenAI Gym compatible environment
```

**Task categories:**

| Category | Count | Examples |
|----------|-------|---------|
| Click | 25+ | `click-button`, `click-dialog`, `click-menu`, `click-option` |
| Form filling | 20+ | `login-user`, `enter-text`, `enter-date`, `enter-password` |
| Navigation | 15+ | `navigate-tree`, `use-slider`, `click-tab-2` |
| Search | 10+ | `search-engine`, `click-link` |
| Multi-step | 10+ | `book-flight`, `social-media`, `terminal` |
| Drag & drop | 5+ | `drag-box`, `drag-shapes`, `drag-sort-numbers` |
| Visual | 5+ | `count-shape`, `identify-shape` |

### Task Format

```javascript
// Each task is a standalone HTML file
// Example: click-button.html
/*
 * Task description: Click the button that says "Submit"
 * Reward: +1 if button clicked, 0 otherwise
 * Time limit: 10 seconds
 */

// The gym environment wraps each HTML task
env = gym.make("MiniWoBClickButton-v0")
obs = env.reset()
# obs["utterance"] = "Click the button that says 'Submit'"
# obs["screenshot"] = numpy array (160x210 pixels)
# obs["dom"] = DOM tree (for text-based agents)
```

---

## 3. Environment Implementation

### OpenAI Gym Interface

```python
import gym
import miniwob
from miniwob.environment import MiniWoBEnvironment

# Create environment
env = MiniWoBEnvironment("click-button")
env.configure(num_instances=1, headless=True)

# Reset and get observation
obs = env.reset()
utterance = obs["utterance"]  # "Click the button that says 'OK'"
screenshot = obs["screenshot"]  # 160x210 numpy array
dom = obs["dom"]              # DOM tree

# Take action
action = create_action("click", coords=(80, 105))
obs, reward, done, info = env.step(action)
print(f"Reward: {reward}")   # 1.0 = success, 0.0 = failure

env.close()
```

### Action Space

```python
# Two action modalities
from miniwob.action import MiniWoBAction

# Coordinate-based (vision approach)
action = MiniWoBAction(action_type="click", coords=(x, y))
action = MiniWoBAction(action_type="type", text="hello")

# Element-based (DOM approach)
action = MiniWoBAction(action_type="click_element", element=dom_element)
action = MiniWoBAction(action_type="focus_element", element=input_field)
```

### Observation Space

```python
obs = {
    "utterance": "Click the button that says 'OK'",
    "screenshot": np.array(shape=(160, 210, 3)),  # RGB screenshot
    "dom": {  # DOM tree (for text-based agents)
        "tag": "div",
        "children": [...],
        "left": 0, "top": 0, "width": 160, "height": 210,
    }
}
```

---

## 4. MiniWoB++ vs Modern Benchmarks

| Feature | MiniWoB++ | Mind2Web | WebArena | WebVoyager |
|---------|-----------|----------|----------|------------|
| Year | 2017/2018 | 2023 | 2023 | 2024 |
| Websites | Simulated | 137 real | 5 self-hosted | 15 live |
| Tasks | 104 | 2,350 | 812 | 643 |
| Execution | Required | Snapshot only | Required | Required |
| Reset | Instant | N/A | Docker restart | N/A |
| Cost | Free | Free | Free | $$$LLM |
| Speed | Fast (~1s/task) | Batch offline | Medium | Slow |
| Reproducibility | Perfect | Good | Good | Poor (live sites) |

---

## 5. Performance Benchmarks (Historical)

| Year | Method | Score (avg across tasks) |
|------|--------|--------------------------|
| 2017 | WoB baseline (RL) | ~16% |
| 2018 | Workflow-Guided RL | ~24% |
| 2020 | CC-Net (ICLR) | ~37% |
| 2022 | WebN-T5 | ~45% |
| 2023 | RCI Agent (GPT-3.5) | ~64% |
| 2023 | WebAgent (Flan-U-PaLM) | ~69% |
| 2024 | GPT-4V + SoM | ~70%+ |

Human performance: ~78% (some tasks require domain knowledge)

---

## 6. Repository Structure

```
MiniWoB/
├── html/
│   └── miniwob/
│       ├── *.html           # 104 task HTML files
│       └── assets/          # CSS, JS dependencies
├── python/
│   └── computergym/
│       ├── miniwob/
│       │   ├── miniwob_env.py       # Gym environment wrapper
│       │   └── registration.py     # gym.make() registration
│       └── setup.py
└── README.md
```

---

## 7. Setup and Usage

```bash
# Install
git clone https://github.com/stanfordnlp/miniwob-plusplus
pip install -e .
pip install selenium  # or playwright

# Quick test
python -c "
import gym, miniwob
env = gym.make('MiniWoBClickButton-v0')
obs = env.reset()
print(obs['utterance'])  # → 'Click the OK button'
env.close()
"
```

---

## 8. Significance and Current Role

MiniWoB++ shaped the entire web agent research paradigm:
- First benchmark for RL-based web automation
- Established the screenshot + DOM observation space
- Enabled rapid prototyping (tasks complete in <1 second)
- Still used for: ablation studies, fast iteration, curriculum training

**Why not used for SOTA comparison anymore:**
- Simulated sites don't reflect real-world complexity
- Task set too narrow (104 tasks, limited diversity)
- Most modern LLM agents achieve near-saturation on easy tasks
- No real-world generalisation signal

**Still valuable for:** debugging agent logic, testing action space formulations, curriculum learning, regression testing.

**License:** MIT  
**Repository:** https://github.com/Farama-Foundation/miniwob-plusplus (maintained fork)  
**Original:** https://github.com/stanfordnlp/miniwob-plusplus  
**Snapshot date:** 2026-03-18
