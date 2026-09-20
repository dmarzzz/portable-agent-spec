---
name: review-rust
description: Checklist for reviewing a Rust diff. Use when the change touches .rs files.
---

# Review a Rust diff

1. List changed `.rs` files with `git diff --name-only`.
2. Search the changed lines for `unwrap(`, `expect(` and `unsafe` with `rg`.
3. For each hit, read 20 lines of context before deciding whether it is a finding.
4. Write each finding as: file, line, one sentence on the risk, one sentence on the fix.
