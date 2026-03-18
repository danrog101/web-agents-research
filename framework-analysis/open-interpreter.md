# Open Interpreter Research Report

## Executive Summary

Open Interpreter is a **natural language interface for computers** — not a web-specific agent but a general-purpose code-executing agent. Users describe tasks in natural language; the LLM generates Python, JavaScript, or Shell code; the code executes locally; results feed back for the next iteration. Browser automation is one capability among many (alongside file manipulation, data analysis, API calls). It was one of the first open-source projects to demonstrate that LLMs could autonomously drive a computer.

---

## 1. Architecture Overview

### Core Philosophy

**Code-as-actions:** Instead of a predefined action space (click, type, scroll), Open Interpreter gives the LLM unlimited expressiveness — it can write any code to accomplish any goal. This generality comes at the cost of predictability and safety.

**Local execution:** Everything runs on the user's machine. No cloud infrastructure required. Full internet access, no file size limits, no runtime restrictions — the exact opposite of hosted sandboxes.

### Package Structure

```
open-interpreter/
├── interpreter/
│   ├── core/
│   │   ├── core.py              # Main Interpreter class
│   │   ├── computer/
│   │   │   ├── computer.py      # Computer abstraction
│   │   │   ├── terminal/        # Code execution environments
│   │   │   │   ├── python.py    # Python runtime
│   │   │   │   ├── javascript.py
│   │   │   │   └── shell.py
│   │   │   ├── browser/         # Browser control (Selenium)
│   │   │   ├── files/           # File system operations
│   │   │   ├── vision/          # Screenshot analysis
│   │   │   └── display/         # GUI control
│   │   └── utils/
│   ├── terminal_interface/
│   │   └── terminal_interface.py  # CLI chat interface
│   └── __init__.py
├── docs/
└── pyproject.toml
```

---

## 2. Core Components Deep Dive

### 2.1 Interpreter Class (`interpreter/core/core.py`)

```python
from interpreter import interpreter

# Basic chat
interpreter.chat("Plot AAPL stock price vs META for the last year")
# → LLM generates Python + matplotlib code
# → Executes locally
# → Shows chart

# Single command
interpreter.chat("Create a PDF report of top 5 news stories today")
# → LLM writes web scraping + PDF generation code
# → Executes

# Programmatic use
result = interpreter.chat("What files are in ~/Documents?", display=False)
# Returns: [{"role": "assistant", "type": "message", "content": "..."}]
```

**Key configuration:**

```python
from interpreter import interpreter

# LLM configuration (via LiteLLM)
interpreter.llm.model = "gpt-4o"
interpreter.llm.api_key = "sk-..."
interpreter.llm.temperature = 0.0

# Local model
interpreter.offline = True
interpreter.llm.model = "openai/x"
interpreter.llm.api_base = "http://localhost:1234/v1"
interpreter.llm.api_key = "fake_key"

# Safety settings
interpreter.auto_run = True   # Execute without confirmation
interpreter.safe_mode = "auto"  # "off", "ask", "auto"

# Vision
interpreter.llm.supports_vision = True
```

### 2.2 Computer Object (`interpreter/core/computer/computer.py`)

```python
# The Computer class wraps all local capabilities

computer = interpreter.computer

# Code execution
computer.run("python", "import pandas as pd; df = pd.read_csv('data.csv')")
computer.run("javascript", "console.log(process.version)")
computer.run("shell", "ls -la")

# Browser (Selenium)
computer.browser.search("open-source web agents")
computer.browser.go_to("https://github.com/browser-use/browser-use")

# Files
computer.files.read("report.md")
computer.files.write("output.txt", content)

# Display/GUI
computer.display.screenshot()
computer.mouse.click(x, y)
computer.keyboard.write("Hello World")
```

### 2.3 LiteLLM Integration

Open Interpreter uses LiteLLM for universal model access:

```python
# 100+ models via unified interface
interpreter.llm.model = "gpt-4o"           # OpenAI
interpreter.llm.model = "claude-3-5-sonnet"  # Anthropic
interpreter.llm.model = "gemini-2.5-pro"    # Google
interpreter.llm.model = "ollama/llama3"     # Local via Ollama
interpreter.llm.model = "azure/gpt-4o"     # Azure
interpreter.llm.model = "bedrock/..."      # AWS
```

### 2.4 Execution Flow

```
User: "Go to github.com and find the most starred Python project this month"
    ↓
LLM generates code:
    from selenium import webdriver
    from selenium.webdriver.common.by import By
    
    driver = webdriver.Chrome()
    driver.get("https://github.com/trending/python?since=monthly")
    
    first_repo = driver.find_element(By.CSS_SELECTOR, ".Box-row h2 a")
    print(f"Top repo: {first_repo.text}")
    print(f"URL: {first_repo.get_attribute('href')}")
    driver.quit()
    ↓
Code executes locally
    ↓
Output: "Top repo: pytorch/pytorch  URL: https://github.com/pytorch/pytorch"
    ↓
LLM continues conversation with result
```

---

## 3. Architecture Diagram

```
User (CLI / Python)
    ↓
Interpreter Class
    ├─ Conversation manager
    └─ LLM (via LiteLLM — 100+ models)
        → Generates code (Python / JS / Shell)
        ↓
Computer Object
    ├─ Terminal runners
    │   ├─ Python runtime
    │   ├─ JavaScript (Node.js)
    │   └─ Shell/Bash
    ├─ Browser (Selenium WebDriver)
    ├─ Files (read/write)
    ├─ Display (screenshot, mouse, keyboard)
    └─ Vision (screenshot analysis)
```

---

## 4. Key Technical Decisions

**Why code generation instead of predefined actions?**
Maximum expressiveness — any task is possible if the LLM can write code for it. No need to pre-define an action schema. The trade-off is unpredictability and security risk.

**Why LiteLLM?**
Supports 100+ models with a unified interface. Switching from GPT-4o to a local Ollama model is a single line change.

---

## 5. Limitations

- **Security risk** — LLM-generated code executes locally with full permissions; could delete files, exfiltrate data
- **Non-deterministic** — same prompt may generate different code on different runs
- **Selenium-based browser** — dated compared to modern CDP/Playwright approaches
- **Not a web-automation framework** — browser is one tool among many; purpose-built web agents outperform it on browser tasks

**License:** AGPL-3.0  
**Language:** Python 3.10-3.11  
**Repository:** https://github.com/openinterpreter/open-interpreter  
**GitHub stars:** ~60k  
**Snapshot date:** 2026-03-18
