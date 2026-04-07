# browser-use — Setup Guide

## Requirements

- Python 3.14 (or 3.10+)
- Groq API key (free tier)
- Windows 10/11

## Installation

```cmd
pip install browser-use
playwright install chromium
```

## Configuration

Create a `.env` file in your project directory:

```
GROQ_API_KEY=your_groq_api_key_here
```

## Agent Script

Create `agent.py`:

```python
import asyncio
import os
from dotenv import load_dotenv
from langchain_groq import ChatGroq
from browser_use import Agent

load_dotenv()

async def main():
    agent = Agent(
        task="Go to https://books.toscrape.com. Find the book called 'Soumission' and note its price and star rating. Then navigate to the Mystery category and find the most expensive book. Report the title, price, and star rating of the most expensive Mystery book.",
        llm=ChatGroq(
            model="meta-llama/llama-4-scout-17b-16e-instruct",
            api_key=os.getenv("GROQ_API_KEY")
        ),
    )
    result = await agent.run()
    print(result)

asyncio.run(main())
```

## Running

```cmd
python agent.py
```

## Notes

- No database required
- No Docker required
- Free Groq API tier is sufficient for most runs
- Rate limiting may cause occasional failures on the free tier
