# Portable Agent

**A directory format for moving a personal agent between harnesses, and for handing it, sealed, to a machine you do not control.**

| | |
|---|---|
| **Status** | Draft spec v0.1. No implementation. Not yet reviewed. |
| **Spec** | [SPEC.md](SPEC.md) |
| **Example** | [example/code-reviewer](example/code-reviewer) |
| **Maintainer** | [@dmarzzz](https://github.com/dmarzzz) |
| **Last update** | 2026-09-19 |
| **License** | CC BY 4.0 |

## The design in one screen

```
my-agent/
  plugin.json  skills/  mcp.json          Agent Plugins 1.0.0 + Agent Skills, unchanged
  AGENTS.md  SOUL.md  USER.md  MEMORY.md  file names harnesses already read
  agent.yaml                              NEW: model requirements, secret names,
                                               privacy labels, port checks
  checks/                                 NEW: the test that says the port worked
  agent.lock                              NEW: sha256 per file, one package hash
```

Grab the folder from one harness, plug it into another. The package states what it needs from a model and never names one, carries no credentials, labels what is private, and brings its own test. Transcripts, channel wiring and schedules stay behind with the installation.

## Open questions

Eight, listed in [SPEC.md section 12](SPEC.md#12-not-specified-yet). The two that matter most for review: whether this should be a proposal to OpenGAP instead of a separate format (OQ-2), and whether permissions can be portable at all (OQ-4).
