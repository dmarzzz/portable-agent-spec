# Portable agent

Status: draft 0.1, 2026-09-19. For review. Nothing here is implemented.

A portable agent is a folder. It holds what makes a personal agent yours (instructions, persona, skills, tools, memory, what it needs from a model) and nothing that belongs to the program running it. Export the folder from one harness, import it into another, and the agent should behave the same. The same folder, encrypted, is what you hand to a machine you do not control.

Most of the folder already has a standard. This spec names those standards, and adds one file for what none of them cover.

## The folder

```
my-agent/
  plugin.json       required  Agent Plugins 1.0.0 manifest: name, version, description
  skills/<name>/SKILL.md      Agent Skills, unchanged
  mcp.json          optional  Agent Plugins MCP config, unchanged
  AGENTS.md         required  operating instructions, plain Markdown
  SOUL.md           optional  persona, tone, boundaries
  USER.md           optional  who the human is                  private by default
  MEMORY.md         optional  curated long-term memory          private by default
  memory/           optional  notes, one Markdown file per fact  private by default
  agent.yaml        required  the agent layer (below)
  agent.lock        generated sha256 of every file, and of the package
```

A portable agent is a valid [Agent Plugin](https://agent-plugins.org): any client that reads Agent Plugins gets the skills and the tools with no extra work. `AGENTS.md` follows [agents.md](https://agents.md). `SOUL.md`, `USER.md` and `MEMORY.md` follow the OpenClaw workspace, which Hermes already imports. Harness-specific settings go where Agent Plugins puts them, under a reverse-domain key in `plugin.json` `extensions`, and every other harness ignores them.

## agent.yaml

What no existing format carries: model requirements, secret names, privacy labels, and a test that the port worked.

```yaml
spec: portable-agent/0.1
instructions: [AGENTS.md, SOUL.md]      # load order

model:
  needs:                                # requirements, never a vendor model id
    context: 64000                      # tokens, minimum
    tool_call: true                     # key names follow models.dev
    structured_output: true
    reasoning: false
  roles: [main, small]                  # small = cheap model for subagents
  tested_on: [qwen3-coder-30b]          # informative only

tools:
  allow: [search_code, read_file]       # subset of mcp.json; empty means all
  needs_bins: [git, rg]

secrets:                                # names and purpose, never values
  - name: GITHUB_TOKEN
    used_by: mcp.json
    required: false

memory:
  load: index                           # index | all | none

privacy:                                # every path is private unless listed
  public: [plugin.json, agent.yaml, checks/]

checks:
  path: checks/
  pass: 0.8
```

Rules:

1. A package never contains a credential. `secrets` lists names, and the importing harness binds each name to its own store.
2. `model.needs` states capabilities. The harness picks a model that meets them and refuses the import if none does.
3. A harness that cannot support a section says so at import and continues. It does not guess.
4. Unknown keys are ignored. New keys are optional. A key never changes meaning within a major version.

## Memory

`MEMORY.md` is the curated file every harness in scope already understands. `memory/` is optional detail: one Markdown file per fact with frontmatter `name`, `description`, `type` (`user`, `feedback`, `project`, `reference`) and optional `updated`. A harness with its own store imports these and exports back to this shape. Memory is private by default and is never loaded in a shared or group context.

## Port checks

A port usually fails at the model swap, and nobody notices. So the package carries its own test. `checks/checks.json` uses the case shape from the Agent Skills eval guide: `prompt`, `files`, `assertions`.

```json
[{ "prompt": "Review this diff.",
   "files": ["checks/fixtures/unwrap.diff"],
   "assertions": ["mentions unwrap or panic", "reports at most 5 findings"] }]
```

After import, the harness runs the cases on the model it chose and reports the pass rate against `checks.pass`. A port is done when the checks pass.

## Not in the folder

Session transcripts and logs. Credentials. Channel and gateway wiring (Matrix, Signal, Slack). Schedules and heartbeats. Harness UI settings. These belong to an installation, and a second installation should not inherit them.

## Running it on a machine you do not trust

Pack the folder as a deterministic tar, encrypt it, and publish the ciphertext beside the `public` paths in clear. `agent.lock` gives the package hash that a contract or a receipt can name. The runner is a public program inside attested hardware. It imports the package the way any harness would, after your key holder releases the key to it. The package says what it needs from a model because the runner decides which attested model serves it, and the package cannot. Run the port checks against the models on offer before you publish.

## Conformance

| level | a harness can |
|---|---|
| 1 | import and export `plugin.json`, skills, `mcp.json` and instructions |
| 2 | level 1, plus `secrets` binding, `tools.allow` and `model.needs` |
| 3 | level 2, plus memory and port checks |

## Prior art

Checked 2026-09-19. Agent Plugins 1.0.0 lists permissions, provenance, secrets and testing as future work, and has no persona, memory or model content. OpenClaw and Hermes each ship a one-way importer from the other, and neither exports to a neutral format or checks the result. Hermes profile distributions never carry memory. OpenGAP (`agent.yaml` plus `SOUL.md`) is the closest existing format: it names concrete models and has no secrets, signing or port checks. No surveyed format states model requirements.

## Open questions

1. Reverse-domain namespace for `plugin.json` `extensions`. Agent Plugins says to base it on a domain the author controls. Not chosen yet.
2. Fold this into OpenGAP's `agent.yaml` as a proposal there, or keep it separate?
3. Is a permissions section portable at all? Harnesses disagree completely (tool allowlists, path trust, human approval).
4. Should schedules be in scope as declarations ("every Monday 08:00, run this prompt")?
5. Signing: `agent.lock` plus a detached signature, or the JCS plus JWS recipe A2A uses for agent cards?
