---
inclusion: always
---
# Product: LockKeeper 1914

## What it is
In 1914 the Panama Canal locks were operated from control boards built by General Electric. Brass interlocking bars under each board made it physically impossible to move the control handles in a dangerous order. LockKeeper resurrects that mechanism in software. It has two parts:

1. **The 1914 board.** An interactive reconstruction of one lock flight, driven by interlocking rules transcribed from the January 1914 General Electric Review. An exhaustive checker proves that no dangerous state can be reached under those rules. When the user removes an interlock, the checker shows the shortest sequence of moves that now leads to disaster.
2. **The modern use.** The same engine guards an AI agent working on cloud infrastructure, so it cannot run valid commands in the wrong order. One example is deleting a database before its backup has been verified.

## Who it is for
- Platform and SRE teams that let agents or automation run multi-step operations in production.
- Anyone curious about how the 1914 Canal was operated safely.

## What it is NOT
- Not a hydraulic or physics simulation of the Canal. Water is modeled only as connectivity.
- Not a complete digital twin of Gatun, Pedro Miguel or Miraflores. It models one representative lock flight.
- Not an LLM-based guardrail. No language model decides whether an action is allowed.
- Not affiliated with or endorsed by the Panama Canal Authority. It is an independent educational and engineering project.

## Tone
Precise, historically honest, quietly dramatic. The drama comes from real history, not invented claims.
