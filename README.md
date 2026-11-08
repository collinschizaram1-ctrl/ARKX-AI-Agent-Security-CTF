# ARKX AI Agent Security CTF — Writeups

Detailed, teaching-style writeups from [ARKX](https://ctf.arkx.ninja) — a capture-the-flag platform built entirely around attacking AI agents. Where most CTFs test one skill at a time, ARKX deliberately mixes two kinds of challenge in the same set of levels: pure AI-layer attacks (prompt injection, jailbreaking, tool abuse) and classic application-security bugs (SSRF, deserialization, race conditions, broken access control) that just happen to live behind a chat interface instead of a traditional API.

That mix is the whole point of this repo. A recurring pattern across almost every level here: **the AI model itself usually isn't the weak point. The infrastructure wired up behind it is.** File systems, session state, shared trust between agents, financial logic, network access — the model is often just the unusual, English-shaped front door into a very ordinary backend bug.

Each writeup walks through the full exploitation chain step by step — the actual prompts and payloads used, the reasoning behind each move, the dead ends that mattered, and the real root cause once the flag is in hand. Nothing is skipped or summarized away; the goal is for someone with no prior context to be able to follow the whole thing and actually understand *why* it worked, not just copy commands.

---

## About ARKX

ARKX organizes its challenges into difficulty tracks:

- **Advanced Track** — classic web/application vulnerabilities reached through an AI agent's tool-calling layer (SSRF, filter bypasses, backend protocol exposure)
- **Expert Track** — deeper, often multi-step chains involving unsafe deserialization, multi-agent trust boundaries, and application logic flaws like race conditions

Levels are deliberately standalone — each one targets a distinct vulnerability class, so solving one doesn't give you much of a head start on the next. That's reflected in how different these writeups end up reading from each other, even though the surface-level format (an AI chatbot you talk to) stays the same throughout.





<img width="1892" height="976" alt="image" src="https://github.com/user-attachments/assets/7b8621a7-91a6-4454-9bf1-94aa706a40f0" />

---

## Levels in This Repo

| # | Level | Track | Vulnerability Class | Status |
|---|---|---|---|---|
| 1 | [Bad Robot](./01-bad-robot.md) | Advanced | Insecure Output Handling — error-channel exfiltration from an `eval()`-based "calculator" | ✅ Solved |
| 2 | [The Shape Shifter](./02-shape-shifter.md) | Expert | Unsafe YAML Deserialization — remote code execution via `!!python/object/apply` tags | ✅ Solved |
| 3 | [The Octopus](./03-octopus.md) | Expert | Multi-Agent Trust Escalation — privilege state shared across agents with no identity verification | ✅ Solved |

More levels will be added as they're documented.

---

## How to Read These

Each writeup follows the same shape:

1. **TL;DR** — the vulnerability in one or two sentences, for skimming
2. **The Setup** — what the level presents itself as, and why that framing matters
3. **Step-by-step exploitation** — every payload used, in order, with the reasoning behind each move explained *before* the result, not just after
4. **Root cause** — why the vulnerability actually exists at a design level, not just what input triggered it
5. **Impact** — what this would mean in a real, production deployment of a similar system
6. **Mapping** — the relevant [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/) category and [MITRE ATLAS](https://atlas.mitre.org/) technique

A pattern worth noticing across all three levels so far: almost every breakthrough came from **reading error messages carefully instead of treating them as noise.** Systems that fail "safely" often leak exactly what they expect next, right there in the rejection. That habit — slow down, read the whole error, change one variable at a time — matters more than any specific payload in this repo.

---

## A Note on Technique, Not Just Flags

These writeups try to explain the underlying idea behind each exploit, not just the specific commands that happened to work on this particular platform. A flag is proof a bug exists; it isn't the finding itself. Where possible, each writeup also notes how the same vulnerability class shows up in real-world systems outside of CTFs — because the actual value of doing this kind of work is building the instinct to recognize these patterns anywhere, not memorizing one platform's specific payloads.

---

**Author:** [Nome / Kholinz](https://x.com/kholinz_sec) — AI security researcher, bug bounty hunter.
