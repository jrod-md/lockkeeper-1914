---
inclusion: always
---
# Tech stack and structure

## Stack
- TypeScript everywhere (strict mode).
- `engine/`: pure TypeScript with no UI or Node-specific dependencies. Shared by the web app, CLI, MCP server and tests.
- `app/`: React + Vite. The board is drawn in SVG (2D). Three.js is NOT allowed unless the developer explicitly approves it.
- `cli/`: Node CLI exposing `verify` (exhaustive check of all rule sets).
- `mcp/`: Node MCP server (stdio) wrapping the engine.
- Tests: Vitest + fast-check (property-based tests).
- Deployment: static site. No backend and no database.

## Layout
```
engine/   state, rules, transitions, checker
rules/    canal-1914 and cloud rule sets (declarative data)
app/      web UI
cli/      verify command
mcp/      MCP server
docs/     MODEL.md, SOURCES.md, BITACORA.md
.kiro/    steering, specs, hooks
```

## Engine design (decided, do not change without asking)
- State = control handle positions, encoded compactly (bitmask or similar).
- Rules are declarative data. Constraint primitives: `precondition`, `atMost(k, set)`, `exclusion`, `conditional`, `key`.
- Hazards are structural predicates (water connectivity), not fluid simulation.
- Checker = breadth-first search over reachable states. It returns either "0 hazardous states reachable out of N" or the SHORTEST counterexample path.
- The checker must be able to run in a Web Worker.

## Conventions
- Small, pure functions. No hidden global state in the engine.
- Every rule has a regression test: disabling it must make a specific hazard reachable. If disabling a rule produces no hazard, flag it.
- Don't add dependencies without saying why.
