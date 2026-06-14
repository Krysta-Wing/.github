# Krysta Wing
### The Execution and Observability Layer for AI Agent Systems.
Traditional application infrastructure was built for predictable human input. AI agents change everything. They generate untrusted code, execute complex loops, and require stateful runtime environments.

Krysta Wing is the missing infrastructure layer that sits between autonomous AI agents and the damage they could do. We provide ultra-low latency, sandboxed, and stateful workspaces that ensure agent code runs safely, streams in real time, and never propagates breaking changes downstream.

---

###  The Problem We Solve
When AI agents write and execute code, engineering teams face a broken choice:
- The Request/Response Box: Submit code to a heavy, black-box micro-VM, wait, and get a raw log dump. If the agent writes an infinite loop, your user interface completely freezes.
- The Stateless Reset: Use a basic container that wipes its memory after every single line of code. If your agent creates a file in Step 1, it vanishes before Step 2.
- The Disaster: Execute code directly on bare host servers—leaving the company one bad LLM generation away from a catastrophic database wipe or a data exfiltration leak.
Krysta Wing fixes this by introducing the three core primitives of autonomous runtime safety.

---
### Core Primitives: Execute → Observe → Validate
Every component we build maps to three critical production requirements:
```
    [ EXECUTE ]             ──►             [ OBSERVE ]             ──►            [ VALIDATE ]
Run untrusted code in                   Log every stdout chunk,                  Enforce structural 
isolated Docker environments.           memory metric, and event                 rules and metrics before
Streams live via SSE.                   to a structured jobId trace.             passing data downstream.
```

- Execute (noa-daemon): A lightweight backend service that spawns highly restricted, sandboxed container environments (--network none, hard memory limits) to isolate code execution entirely from host systems.
- Observe (krysta-library): A developer-first Python SDK that streams granular stdout and execution traces live into your agent UI as they happen, eliminating UI freezes.
- Validate (gate.py): Structural and security gates that evaluate execution results against safety policies and schemas before the output is allowed to touch your next agent turn.

###  Developer API Surface

```py
from krysta import Noa

# 1. Spawn a secure, stateful runtime session
async with Noa.spawn(runtime="python", session_id="agent_turn_4") as sandbox:

    # 2. Live streaming — observe stdout chunks as they happen
    async for chunk in sandbox.execute(agent_generated_code):
        print(chunk.stdout)

    # 3. Validate structures before passing downstream
    result = sandbox.validate(rules=["valid_json", "no_network_calls"])
    
    # Extract the full execution trace tied to this jobId
    trace = sandbox.trace()
```

