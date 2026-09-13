---
name: sm
description: Use veno's "sm" skill manager (docs at v3n0.top/sm, source at ~/Projects/sm) to install, enable, and project directory-based skill profiles to agent targets. Use when managing skills, profiles, .smtag activation, sm import/update/apply/shell commands, or when symlinks in an agent skills directory need reconciling.
---

# sm — skill manager

sm stores directory-based skill **profiles** and projects the globally activated set to agent target directories via symlinks. It is package-manager agnostic, never invokes Git, and separates inventory from projection: **import/update change the inventory, apply reconciles targets.**

## Mental model

- **Inventory**: `${SM_HOME:-~/.sm}/profiles/<profile>/<skill>/` — each skill is a directory containing `SKILL.md` (+ scripts). Skill contents are opaque.
- **Activation**: `<profile>/.smtag` holds `true`/`false`. Missing `.smtag` initializes to `true`. Hidden entries in a profile are metadata, not skills.
- **Targets**: named entries in `~/.config/sm/config.toml`, each with a persistent `skills_dir` and/or shell adapter. Every target receives the *same* globally active skill set, as symlinks.
- A skill reference is `<profile>/<skill>`.

## Common commands

```bash
sm profiles                    # list profiles
sm profiles new NAME [--disabled]
sm skills [PROFILE...]         # list <profile>/<skill> lines; no args = all profiles
sm targets                     # list configured target names
sm enabled                     # list globally enabled profiles
sm status [-t TARGET]          # target\tskill\tsource\tdestination

sm enable PROFILE...           # write .smtag; never touches targets
sm disable PROFILE...
sm apply [-t TARGET] [-f]      # project enabled skills to targets
sm gc                          # remove unused shell generations

sm import --profile P [--skill S]... [--from DIR] [--create]   # copy skills into profile
sm update --from DIR [--skill S]... [--profile P | --all]      # replace existing same-named skills
sm export [--profile P]... [--skill P/S]... --to DIR           # copy out; never overwrites

sm shell [--target T]... [--profile P]... [--skill P/S]... [-- CMD...]  # temp session skill set
sm adopt TARGET DIR            # register a target's skills_dir
```

## Semantics and gotchas

- **Order matters**: install/import first, then `sm enable`, then `sm apply`. enable/disable never modify targets — apply does.
- **Apply without `-f`** fails if any non-hidden real directory blocks a desired name. With `-f`, those directories are **removed**. Ordinary files, hidden entries, and unrelated external symlinks are preserved unless blocking.
- All selected targets are preflighted before any mutation. If an apply is interrupted (e.g. OS interrupt), some links may already be reconciled — just rerun `sm apply`.
- Enabling a profile that conflicts with another enabled owner (same skill name) fails **before** any tag is written.
- Import/update leave source directories in place. Update only replaces *existing* same-named inventory skills; unknown source dirs are skipped.
- `sm shell` gives temporary per-session sets without changing persistent targets; isolated generations live under `~/.cache/sm/generations/` and are GC'd via leases.
- Use `--dry-run` on any mutation to preview tab-separated operations first.
- Successful mutations are silent; exit 2 = usage error, 1 = validation/operational failure (both occur before inventory/targets change).

## Manual install shortcut

Since profiles are plain directories, a skill can be installed by copying it directly into `~/.sm/profiles/<profile>/<skill>/` — then run `sm enable <profile>` and `sm apply -f` to project it.

## Reference

- Full docs: https://v3n0.top/sm/ (EN + 简体中文)
- Local source and docs: `~/Projects/sm/` (`docs/command-reference.md`, `docs/filesystem-layout.md`)
