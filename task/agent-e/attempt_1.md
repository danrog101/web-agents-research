
# agent-e — Skipped

## Reason

**Windows incompatible** — agent-e requires `uvloop` as a dependency, which does not support Windows.

```
RuntimeError: uvloop does not support Windows at the moment
```

## Installation Attempt

```cmd
git clone https://github.com/EmergenceAI/agent-e.git
cd agent-e
python -m pip install -r requirements.txt
```

## Error

During `pip install -r requirements.txt`, the installation failed on the `uvloop==0.21.0` package:

```
error: subprocess-exited-with-error
× Getting requirements to build wheel did not run successfully.
RuntimeError: uvloop does not support Windows at the moment
```

## System

| Field | Value |
|-------|-------|
| OS | Windows 11 |
| Python | 3.11.9 |
| Framework version | latest (cloned from GitHub) |

## Conclusion

agent-e cannot be installed or run on Windows due to its hard dependency on `uvloop`, a Unix-only asyncio event loop implementation. This framework was therefore excluded from the consistency test.
