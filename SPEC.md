# Portable Agent: draft spec v0.1

Status: draft, 2026-09-19. Normative words (MUST, SHOULD) are used only where the point is settled. Everything else is marked TBD with the open question (OQ) that resolves it. No harness implements this yet.

## 1. Overview

A portable agent is a directory. It holds the definition of a personal agent (instructions, persona, skills, tools, memory, model requirements) and nothing that belongs to an installation. A harness exports the directory, another harness imports it, and a test carried in the directory decides whether the port worked. The same directory, sealed, is the input format for running the agent on a machine its owner does not control.

The directory is a valid Agent Plugin. This spec adds one file, `agent.yaml`, for what no existing format covers (section 10).

## 2. Package

```
struct Package {                          // a directory rooted at one filesystem location
    plugin.json       : PluginManifest    // REQUIRED. Agent Plugins 1.0.0, unchanged
    skills/<n>/SKILL.md : Skill[]         // Agent Skills, unchanged
    mcp.json          : McpConfig         // Agent Plugins 1.0.0, unchanged
    AGENTS.md         : markdown          // REQUIRED. operating instructions (agents.md: plain Markdown, no frontmatter)
    SOUL.md           : markdown          // persona, tone, boundaries
    USER.md           : markdown          // who the human is             label: private
    MEMORY.md         : markdown          // curated long-term memory     label: private
    memory/*.md       : MemoryNote[]      // one fact per file            label: private
    checks/checks.json: Check[]           // port checks (section 6)
    agent.yaml        : AgentManifest     // REQUIRED. section 3
    agent.lock        : Lock              // generated. section 7
}
```

- `name`, `version` and `description` live in `plugin.json` only. `agent.yaml` MUST NOT repeat them.
- Harness-specific settings go under a reverse-domain key in `plugin.json` `extensions` and in the directory of the same name, as Agent Plugins defines. A harness MUST ignore namespaces it does not implement.
- A package MUST NOT contain: credentials, session transcripts, logs, channel or gateway wiring, schedules, harness UI settings. These belong to an installation.

## 3. Manifest

```
struct AgentManifest {
    spec          : "portable-agent/0.1"
    instructions  : path[]                // load order. default [AGENTS.md, SOUL.md]
    model         : {
        needs     : ModelNeeds            // requirements. a vendor model id is never a requirement
        roles     : ("main" | "small" | "embeddings")[]     // default [main]
        tested_on : string[]              // informative
    }
    tools         : {
        allow      : string[]             // subset of tool names from mcp.json. empty = all
        needs_bins : string[]             // executables expected on PATH
    }
    secrets       : { name, used_by: path, required: bool }[]    // names only
    memory        : { load: "index" | "all" | "none" }           // default index
    privacy       : { public: path[] }    // every path not listed is private
    checks        : { path, pass: float } // pass in [0, 1]
}

struct ModelNeeds {                       // key names follow models.dev
    context           : uint              // tokens, minimum
    tool_call         : bool
    structured_output : bool
    reasoning         : bool
    attachment        : bool              // default false
}

struct MemoryNote {                       // YAML frontmatter, then Markdown
    name, description : string
    type              : "user" | "feedback" | "project" | "reference"
    updated           : date              // optional
}
```

- Unknown keys MUST be ignored. New keys MUST be optional. A key MUST NOT change meaning within a major version.
- `secrets[].name` is a name. The importing harness binds it to its own store. A value in a package is a validity failure (section 4).
- Paths labelled private MUST NOT be loaded in a shared or group context, and MUST NOT appear in the public part of a sealed package (section 8).

## 4. Validity

```
def package_valid(P) -> bool:
    if not agent_plugins_valid(P.plugin_json, P.mcp_json):      return False   # their schemas, unchanged
    if P.agent.spec.major != 0:                                 return False
    if any(not exists(P, f) for f in P.agent.instructions):     return False
    if any(t not in tool_names(P.mcp_json) for t in P.agent.tools.allow):  return False
    if any(looks_like_credential(v) for v in env_values(P.mcp_json)):      return False   # heuristic: OQ-6
    if any(is_private_default(p) for p in P.agent.privacy.public):         return False   # USER.md, MEMORY.md, memory/
    if P.lock and P.lock.package_hash != package_hash(P):       return False
    return True
```

## 5. Import and export

```
def import_agent(H, P) -> Report:
    assert package_valid(P)
    R = Report()
    M = H.pick_models(P.agent.model.needs, P.agent.model.roles)
    if M is None: return R.refuse("no model meets model.needs")       # never substitute silently
    for s in P.agent.secrets:
        if not H.bind_secret(s.name) and s.required: return R.refuse(s.name)
    for section in [instructions, skills, tools, memory]:
        if H.supports(section): H.load(section, P)
        else: R.skipped.append(section)                                # say so, do not guess
    R.checks = run_checks(H, M, P)                                     # section 6
    return R

def export_agent(H) -> Package:
    P = H.definition()               # definition only. history stays (section 2)
    P.lock = make_lock(P)
    assert package_valid(P)
    return P
```

`export_agent(import_agent(P))` SHOULD reproduce every file the harness supports byte for byte, and MUST preserve files it does not support.

## 6. Port checks

```
struct Check { prompt: string, files: path[], assertions: string[] }   // case shape from the Agent Skills eval guide

def run_checks(H, M, P) -> { rate: float, ok: bool }:
    passed = sum(all(judge(a, H.run(M, c.prompt, c.files)) for a in c.assertions) for c in P.checks)
    rate = passed / len(P.checks)
    return { rate, ok: rate >= P.agent.checks.pass }
```

A port is complete when `ok`. How `judge` evaluates an assertion (string match, model judge): TBD (OQ-3). A package with no checks imports with `ok = unknown`.

## 7. Lock

```
struct Lock { files: { path: sha256 }, package_hash: sha256 }

def package_hash(P):
    return sha256(concat(path || 0x00 || sha256(bytes) for path, bytes in sorted(P.files) if path != "agent.lock"))
```

Signature over `package_hash`: TBD (OQ-5).

## 8. Sealed form

| Part | Contents | Visible to |
|---|---|---|
| public | every path in `privacy.public`, plus `agent.lock` | anyone |
| sealed | deterministic tar of the whole package, encrypted to a key the owner holds | a runner the owner's key holder has released the key to |

- The runner is a public program that executes `import_agent` inside attested hardware. It is one more harness.
- The runner picks the model, never the package. `model.needs` and the port checks are how an owner learns, before publishing, whether the models a runner offers are good enough.
- `package_hash` is the identifier a contract or a receipt names.
- Executable files in a package (tool scripts) run with whatever isolation the runner provides. This spec makes no claim about them.
- Encryption scheme, key release and attestation are out of scope here.

## 9. Conformance

| Level | A harness implements |
|---|---|
| 1 | `package_valid`; import and export of `plugin.json`, skills, `mcp.json`, instructions |
| 2 | level 1, plus `secrets`, `tools.allow`, `model.needs` with refusal |
| 3 | level 2, plus memory, `run_checks`, `agent.lock` |

## 10. Reused and new

| Component | Source | Changed |
|---|---|---|
| `plugin.json`, `mcp.json`, extension namespaces | Agent Plugins 1.0.0 | no |
| `skills/<n>/SKILL.md` | Agent Skills | no |
| `AGENTS.md` | agents.md | no |
| `SOUL.md`, `USER.md`, `MEMORY.md` | OpenClaw workspace file names (Hermes imports them) | no |
| `ModelNeeds` key names | models.dev | used as requirements, not as facts about a model |
| `Check` case shape | Agent Skills eval guide | moved to package level |
| model requirements, secret names, privacy labels, port checks, lock | this spec | new |

Checked 2026-09-19: Agent Plugins 1.0.0 lists permissions, provenance, secrets and testing as future work and carries no persona, memory or model content. OpenClaw and Hermes each import from the other, one way, with no neutral format and no check of the result. OpenGAP (`agent.yaml` plus `SOUL.md`) is the closest existing format. It names concrete models and has no secrets, lock or port checks. No surveyed format states model requirements.

## 11. Constants

| Name | Value | Decided by |
|---|---|---|
| `SPEC_VERSION` | `portable-agent/0.1` | this draft |
| `HASH` | SHA-256 | this draft |
| default `checks.pass` | 0.8, provisional | OQ-3 |
| extension namespace | TBD | OQ-1 |

## 12. Not specified yet

- OQ-1: reverse-domain namespace for harness extensions. Agent Plugins says to base it on a domain the author controls.
- OQ-2: propose `agent.yaml` to OpenGAP, or keep it separate.
- OQ-3: `judge` semantics and the default pass rate.
- OQ-4: permissions. Harnesses disagree completely (tool allowlists, path trust, human approval), so v0.1 has only `tools.allow`.
- OQ-5: signature over `package_hash`: detached signature, or JCS (RFC 8785) plus JWS as A2A uses for agent cards.
- OQ-6: what `looks_like_credential` is.
- OQ-7: schedules as declarations ("every Monday 08:00, run this prompt"). Out of scope in v0.1.
- OQ-8: memory merge on re-import (supersession, provenance).
