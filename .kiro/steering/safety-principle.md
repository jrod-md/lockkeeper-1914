---
inclusion: always
---
# Safety principle (non-negotiable)

In 1914 an operator's intention could not override a correctly placed interlocking bar. LockKeeper keeps that property:

1. **Allow/deny decisions are made only by the deterministic engine.** No LLM, prompt or heuristic participates in deciding whether an action is permitted.
2. An agent may ask WHY an action was blocked. The engine returns the rule, its source and the required preconditions. An agent may NEVER bypass, disable or weaken a rule.
3. There is no MCP tool, API or flag that lets an agent override a block. Overrides exist only as **keys**: explicit, logged actions taken by a human, modeled on the 1914 chief operator's key (S p.14).
4. Removing an interlock in the demo is an explicit, visible act that always triggers a re-check.
5. If code you are writing would let an agent influence an allow/deny decision, stop and flag it to the developer.
