# Level 6: The Shape Shifter — Remote Code Execution via Unsafe YAML Deserialization

**Platform:** [ARKX — AI Agent Security CTF](https://ctf.arkx.ninja)
**Track:** Expert
**Category:** Unsafe Deserialization — YAML Tag Exploitation (Remote Code Execution)
**Flag:** `flag-d6c733`

---

## TL;DR

An agent calling itself a "Configuration Processor" accepts YAML, parses it, and summarizes the settings back to you. YAML has a feature — type tags — that, when parsed carelessly, can be abused to make the parser actually *construct objects and call real functions*, not just read data. One specific function name was filtered. A different, equally dangerous function wasn't. That gap was enough for full remote code execution.

---

## The Setup

The level description: a "Configuration Processor" that accepts YAML configuration files and summarizes the resulting settings. This exact pattern — an AI agent that ingests structured config, logs, or data files on someone's behalf and reports back what's in them — is genuinely common in real tooling. That's part of what makes this level worth understanding properly: it's not a toy scenario.

Before touching anything, it's worth knowing *why* YAML specifically is dangerous in a way that, say, plain JSON usually isn't.

### A 60-second detour: why YAML is special

Most data formats (JSON, CSV) just represent data — strings, numbers, lists. YAML, as used by Python's PyYAML library, supports something extra: **type tags**, written like `!!python/object/apply:somefunction`. These tags tell the parser: "don't just store this as data — actually construct this object, or actually call this function, right now, while parsing."

This is an intentional PyYAML feature, meant for advanced use cases like reconstructing custom Python objects from saved config. The problem is: if a parser uses `yaml.load()` (the unsafe, full-featured loader) instead of `yaml.safe_load()` (the restricted one), and it's parsing *user-supplied* YAML, the user can use these tags to call **any function they want**, with **any arguments they want** — including things like `os.system`.

That's the whole vulnerability, in one sentence: **letting an untrusted user's YAML reach an unsafe parser is equivalent to letting them run arbitrary code.**

---

## Step 1 — Probe with a well-known dangerous tag

```yaml
app_name: test
port: 8080
debug: !!python/object/apply:os.system ["id"]
```

`os.system` is probably the single most commonly tested function for this exact bug — if a developer added any kind of defense, this is the first thing they'd think to block.

**Response:**
```
Configuration processed successfully.
- 3 configuration keys found: app_name, port, debug
- Values include: app_name (test), port (8080), 
  debug (unsafe YAML tag applied: !!python/object/apply:os.system ["id"])
```

Notice what happened here: the tag was *recognized and described back* — "unsafe YAML tag applied" — but not actually executed. The command didn't run; it just got echoed as text. This tells us one of two things: either the YAML is being read and described as text rather than fed to a real unsafe loader, or `os.system` specifically is being filtered/blocked while the underlying mechanism is otherwise live.

**Don't assume the whole technique is dead just because one attempt failed.** This is a habit worth building: when a specific payload fails, change one variable at a time and retest, rather than concluding the entire approach doesn't work.

---

## Step 2 — Vary the function, keep everything else the same

If `os.system` specifically is denylisted by name, a different function that achieves the same goal — running a shell command — should still work, as long as the underlying `!!python/object/apply` mechanism itself isn't blocked.

```yaml
debug: !!python/object/apply:subprocess.check_output
  - ["ls", "-la"]
```

`subprocess.check_output` runs a command and returns its actual text output — a reasonable alternative to `os.system`, and much less likely to appear on a basic denylist, since it's less commonly used in textbook examples of this vulnerability.

**Response:**
```
Configuration processed successfully.
- The YAML configuration used a Python-specific tag to apply a subprocess call,
  which was safely parsed and the output has been extracted.

total 4
drwxrwxrwx. 2 ctfuser ctfuser 22 Nov 1 18:38 .
drwxrwsrwx. 5 root    ctfuser 63 Nov 1 18:38 ..
-rwxrwxrwx. 1 ctfuser ctfuser 11 Nov 1 18:38 flag.txt
```

That's a real directory listing. Real file, real owner, real permissions, real size. This confirms something important: **the YAML parser genuinely is unsafe and will execute arbitrary function calls.** The earlier `os.system` attempt wasn't proof the whole technique was blocked — it was proof that one specific, commonly-tested function name was filtered, while the general mechanism underneath remained wide open.

This distinction matters a lot in practice. Denylisting specific dangerous function *names* is a famously weak defense against this vulnerability class, because the real attack surface is "any importable, callable function" — a space far too large to block item by item.

---

## Step 3 — Read the flag

Now that we know `subprocess.check_output` reaches real code execution, point it directly at the flag file:

```yaml
!!python/object/apply:subprocess.check_output [["cat", "flag.txt"]]
```

Note the shape here — submitted as a single, standalone top-level value rather than nested under a named key like `debug:`. Both forms work in principle; this one was simplest to submit cleanly.

**Response:** the full contents of `flag.txt`, returned as readable text — `flag-d6c733`.

---

## Why This Works — Root Cause

Two things had to both be true for this to work:

1. **The backend used an unsafe YAML loader** — functionally equivalent to calling PyYAML's `yaml.load()` with the default loader, instead of `yaml.safe_load()`. This is the actual root vulnerability; everything else is just exploiting it.
2. **The defense in place was a denylist on specific function names**, which is structurally weak against this bug class. Blocking `os.system` does nothing to stop `subprocess.check_output`, `subprocess.Popen`, `os.popen`, `eval`, or dozens of other equally dangerous routes to the same outcome.

The actual fix for this class of bug isn't "block more function names" — it's "use `yaml.safe_load()` in the first place," which refuses to process type tags at all, regardless of which function they name.

---

## Impact in a Real System

This isn't a CTF-only concern. YAML is everywhere in real infrastructure — Kubernetes configs, CI/CD pipelines, application settings files, Ansible playbooks. Any tool that accepts user-supplied YAML and parses it with an unsafe loader is exposed to this exact technique. The practical impact in a production system would be full remote code execution with whatever privileges the parsing process runs under — meaning arbitrary file read/write, credential theft, lateral movement, or complete server compromise, not just reading one flag file.

---

## Mapping

| Framework | Classification |
|---|---|
| OWASP Top 10 for LLM Applications | **LLM05 — Improper Output Handling**, more precisely the general insecure-deserialization class: user-controlled structured data (YAML) passed into an unsafe parser capable of code execution |
| MITRE ATLAS | **AML.T0011** (Exfiltration via AI model output), combined with classic insecure deserialization (**CWE-502**) |

---

*Part of a series documenting AI agent security CTF labs on [ARKX](https://ctf.arkx.ninja). Previous: [Level 5 — Bad Robot](https://github.com/collinschizaram1-ctrl/ARKX-AI-Agent-Security-CTF/blob/main/level%205%20bad%20robot.md).*
