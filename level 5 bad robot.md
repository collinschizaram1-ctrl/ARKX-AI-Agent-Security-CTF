# Level 5: Bad Robot — Breaking a "Calculator" with Error-Based Exfiltration

**Platform:** [ARKX — AI Agent Security CTF](https://ctf.arkx.ninja)
**Track:** Advanced
**Category:** Insecure Output Handling / Sandbox Escape via Error-Based Exfiltration
**Flag:** `flag-0c7f54`

---

## TL;DR

A chatbot presents itself as a calculator — you send it math, it sends back a number. Underneath, it's running a real Python `eval()` on whatever you type. The developer tried to block the attack by filtering what *displays* as output — but never filtered what gets written into an *error message*. The flag got exfiltrated in one shot by deliberately triggering a type error and reading the secret out of the exception text.

---

## The Setup

The level description says it's a calculator. You type an expression, it gives you a number back. That's the entire premise — and the entire premise is exactly what we're going to break.

The question to ask immediately, before touching anything: **is this a real programming language running my input, or just a simple math parser pretending to be one?**

That distinction matters enormously. A simple parser that only understands `+`, `-`, `*`, `/` is boring and safe. A real interpreter running raw text as code is a loaded gun.

---

## Step 1 — Fingerprint the engine

Send something that only a *real* Python interpreter would understand — not math, but a genuine Python built-in function:

```
globals(1/1)
```

**Response:**
```
Error: globals() takes no arguments
```

Stop and actually read that. A simple calculator parser has no idea what `globals()` even is — it would either ignore it or throw a generic "invalid expression" error. This response is *specific*. It knows `globals()` is a real function, and it knows exactly how many arguments it expects. That's not a math parser talking. That's real Python, under the hood, actually trying to run your code.

**Lesson:** when you're not sure if something is a real interpreter, throw it a function name from that language that has nothing to do with the tool's stated purpose. If it responds with a *real, specific* error about that function, you've got your answer.










---

## Step 2 — Learn how errors are handled

```
1/0
```

**Response:**
```
Error: division by zero
```

In raw, unprotected Python, dividing by zero crashes the whole script and dumps a full stack trace — file paths, line numbers, the works. Here you got one short, clean sentence. No trace, no internals.

That tells you something important: **the real code is wrapped in a `try/except` block.** Someone anticipated errors happening and built a safety net to catch them and show a cleaned-up message instead of a scary crash.

This is the single most important thing to notice in the whole level — not because it's a weakness yet, but because it tells you *where* the weakness is going to be. Errors aren't blocked. They're just reformatted before you see them. Keep that in your back pocket.

---

## Step 3 — Check if the filesystem is reachable

```
open('flag.txt').read()
```

No crash, no complaint about an undefined function. That silence (or a non-error response) tells you the file exists and is readable by whatever process is running this eval. Python's built-in `open()` function works here — nothing is stopping file access at this stage.

---

## Step 4 — Hit the real wall

Here's the frustrating part: running that `open('flag.txt').read()` doesn't actually *show* you anything on screen. No flag text appears. It's tempting to conclude the read is blocked.

**It isn't.** This is the part worth slowing down on, because misreading this step wastes a lot of time.

What's actually happening: the code **executes successfully** — Python really does open and read the file. What fails is a completely separate piece of the system: the **output formatter**, the part that decides how to display whatever `eval()` returns back to you in the chat window. Since this thing is dressed up as a calculator, its formatter was almost certainly built assuming the result will always be a number. Hand it a string (the file's text), and it doesn't know what to do with that — so it just shows nothing.

**Execution ≠ display.** They are two separate systems, and only one of them is broken here.

Once you see that distinction, the strategy writes itself: stop trying to get the string to display directly. Instead, force the result into something the formatter *does* know how to show — a number.

---

## Step 5 — Leak the length first

```
len(open('flag.txt').read())
```

**Response:**
```
Result: 11
```

`len()` doesn't give you the string — it gives you a *count*, an integer. Integers are exactly what this formatter is built to display, so this sails right through.

Now you know the flag is 11 characters long — and `flag-0c7f54` is, in fact, 11 characters.

This step matters for a reason beyond just the number itself. It's a small example of a pattern called **blind extraction**: you never got the system to directly show you the secret. You asked it an indirect question — "how long is this?" — that it was *willing* to answer, and used that permitted answer to learn something about the thing it wouldn't show you directly. This is the exact same logic behind blind SQL injection, where an attacker who can't read a database directly instead asks a long sequence of yes/no or counting questions until the secret is reconstructed piece by piece.

---

## Step 6 — Get the whole thing in one shot

You *could* now go character by character — but there's a faster move available once you understand how Python errors work.

Python's `int()` function, when you hand it a string that isn't a valid number, throws an error — and crucially, that error **includes the exact string it choked on**, verbatim, as part of the message.

```
int(open('flag.txt').read().strip())
```

**Response:**
```
Error: invalid literal for int() with base 10: 'flag-0c7f54'
```

There it is. The whole flag, sitting right inside the error text.

Think back to Step 2: we established the real `eval()` is wrapped in `try/except`, catching errors and showing a sanitized message instead of a raw crash. That safety net was built to hide scary *stack traces* — file paths, line numbers, internal details. But it was never built to filter the *content* of the exception's own message. And Python's `int()` function writes the invalid input directly into that message, by design, because that's genuinely useful for debugging in normal circumstances.

The developer filtered what gets shown when things succeed. They never thought to filter what gets written when things fail. The system wasn't broken into — it was asked an honest question it was architecturally forced to answer, through a door its own error handling left wide open.

---

## Why This Works — Root Cause

Two separate mistakes stacked on top of each other to create this:

1. **Using `eval()` on raw user input at all.** This is dangerous on its own, independent of anything else. It hands any user a live code execution environment, dressed up behind a "calculator" UI.
2. **Filtering output at the wrong layer.** The display logic correctly blocked raw strings from showing — a reasonable instinct — but that control only covered one specific *success-path output shape*. It left numeric results, and crucially, error messages, completely untouched. A control in the wrong place doesn't remove the underlying danger. It just changes what shape of request you need to send to get around it.

---

## A Note for Next Time

`int()` only works as a one-shot leak because the secret here is non-numeric text — that guarantees the conversion fails. If a secret were purely numeric, this exact trick wouldn't trigger an error at all, and you'd need a different approach:

- **Character-by-character extraction** via something like `ord(value[N])`, reading one character's numeric code at a time
- **Binary leaks via arithmetic**, like `1 / (value == guess)` — this either returns a number (if your guess is wrong and the condition is `False`, i.e. `0`, so you get a divide-by-zero error) or succeeds silently (if your guess is right) — letting you leak one true/false bit per request

General rule worth keeping: **when an error channel is available, always try to force the whole value out in a single request first** (like the `int()` trick here) before falling back to a slower, grinding character-by-character loop. It's not always possible, but it's always worth trying first.

---

## Mapping

| Framework | Classification |
|---|---|
| OWASP Top 10 for LLM Applications | **LLM05 — Improper Output Handling** (primary); touches **LLM01 — Prompt Injection**, since the root access vector is injecting executable instructions into a field meant only for constrained arithmetic |
| MITRE ATLAS | **AML.T0051** (LLM Prompt Injection) combined with classic application-layer code injection via `eval()` (CWE-95) |

---

*Part of a series documenting AI agent security CTF labs on [ARKX](https://ctf.arkx.ninja). Next: [Level 6 — The Shape Shifter](https://github.com/collinschizaram1-ctrl/ARKX-AI-Agent-Security-CTF/blob/main/02-shape-shifter.md).*
