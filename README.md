# one-shots

Self-contained prompts that rebuild a working system from an empty directory.

## What this is

Each file in this repo is a **one-shot**: a single markdown document you paste
into a fresh AI coding session (Claude Code, Cursor, whatever) sitting in an
empty folder, and it produces a functionally-equivalent rebuild of a real
system — no access to the original source, no follow-up questions.

Think of it as **lossy compression for software**. The code is thrown away;
what survives is the spec, the interface contracts, and the scar tissue.

## Why

- **Systems as intent, not artifacts.** A 3,000-line codebase distilled to a
  400-line spec forces you to say what actually matters. Everything else was
  incidental.
- **Survives bitrot.** Frameworks die, runtimes change, dependencies vanish.
  A one-shot regenerates against whatever stack is current when you run it.
- **The invariants are the value.** Every good one-shot has a section of
  hard-won lessons — the `KillMode=process`, the 25s heartbeat, the "never
  reappend the DOM node on sort" — that took days to discover and one line to
  state. That section is worth more than the code.
- **Onboarding that actually works.** Hand a new engineer the one-shot instead
  of the repo. They'll understand the system better in an hour than a week of
  reading source.
- **It's a fun party trick.**

## What makes a good one-shot

A one-shot is **not** "please build me a todo app." It is a precise spec for a
system that already exists and works. The bar:

1. **Deliverables tree** — exact file layout the AI should produce.
2. **Hard interface contracts** — verbatim. Schema DDL, wire protocols, env
   vars, regexes, config file contents. Don't describe the SQLite schema;
   paste it. Don't say "a websocket protocol"; say "first byte 0x01 = resize,
   next 4 bytes = cols/rows uint16 LE."
3. **Per-component behavior spec** — for each file: what it does, what it
   calls, what state transitions it owns. Route signatures. Function
   signatures where they're load-bearing.
4. **Non-obvious invariants** — its own section. Every "we tried X, it broke
   because Y, do Z instead." This is the part an AI can't derive from first
   principles and a reader can't get from `git log`.
5. **A closing build instruction** — one paragraph telling the AI what order
   to build in and how to know it worked.

Loose narrative ("the supervisor watches agents and nudges them") is fine as
connective tissue, but every load-bearing detail must be nailed down.

## Anti-patterns

- **Pasting the source code.** That's a backup, not a one-shot. Compress.
- **"Implement reasonable error handling."** Say which errors, or say nothing.
- **Referencing the original repo.** The reader has an empty directory.
- **Skipping the ugly parts.** The retry loop with the magic sleep, the regex
  that took six tries — those are exactly what to keep.
- **Untested.** If you haven't pasted it cold into an empty dir and gotten a
  working system back, it's a draft.

## Contributing

PRs welcome. One file per system: `<NAME>-ONESHOT.md` at the repo root.

**Checklist before opening a PR:**

- [ ] Pasted into a fresh AI session in an empty directory → produced a
      working system on the first try (minor fixups allowed; rewrites are not)
- [ ] Has a deliverables tree
- [ ] Has a non-obvious-invariants section (if yours is empty, your system is
      either very young or you haven't been paying attention)
- [ ] No links to private source, internal wikis, or the original repo
- [ ] One-line entry added to the index below

Partial / WIP one-shots go in `drafts/`. Improvements to existing one-shots
are encouraged — especially adding invariants you hit that the original author
missed.

## Index

| One-shot | Rebuilds | Lines | Stack |
|---|---|---|---|
| [HERDER](./HERDER-ONESHOT.md) | Multi-agent tmux supervisor + web herder UI | ~470 | node/express/ws/xterm + python/asyncio + sqlite + systemd |

## Suggested template

```markdown
# <name> — one-shot rebuild spec

<Two sentences: what the system does and why it exists.>

You are in an empty directory. Build the following from this spec alone.

## Deliverables
    <tree>

## Hard interface contracts
    <env vars, schema DDL, wire protocols, config files — verbatim>

## Component: <file>
    <behavior, signatures, state it owns>
    ...repeat per component...

## Non-obvious invariants
    <numbered list of scar tissue>

## Build order & smoke test
    <what to build first, how to know it works>
```

## License

Each one-shot is licensed by its contributor (note it in the file header).
Default for files without a header: CC-BY-4.0 — reuse freely, credit the
author.
