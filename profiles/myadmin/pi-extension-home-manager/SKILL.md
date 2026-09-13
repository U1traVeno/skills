---
name: pi-extension-home-manager
description: Maintain Veno's Pi extensions and Pi configuration in ~/dotfiles through Home Manager, including local extension creation, keybindings, package decisions, validation, pi-sync activation, and coordination with sm-managed skills. Use when adding, changing, debugging, distributing, enabling, or removing Pi extensions or when editing modules/programs/pi-agent.nix, pi/settings.json, pi/extensions, or the Pi skill links managed by sm.
compatibility: Veno's macOS and Fedora Home Manager outputs, Pi, and sm-skill-manager.
---

# Pi Extension And Home Manager Workflow

Use this skill for Veno's personal Pi customization. Treat `~/dotfiles/AGENTS.md`
as authoritative and read it before making repository changes.

## Ownership Model

Keep each kind of state with its actual owner:

- `~/dotfiles/pi/extensions/*.ts` is the source of truth for personal Pi
  extensions shared between Veno's machines.
- `~/dotfiles/modules/programs/pi-agent.nix` installs out-of-store symlinks into
  `~/.pi/agent/`. It is imported by both `veno@macbook` and `veno@thinkpad`.
- `~/dotfiles/pi/settings.json` and `pi/models.json` are shareable manifests.
  Home Manager links them into `~/.pi/agent/`.
- Pi credentials, npm authentication, package caches, sessions, and generated
  state stay machine-local. Never inspect or commit credentials.
- `sm` owns managed skill symlinks under `~/.agents/skills` and
  `~/.pi/agent/skills`. Home Manager must not create colliding links.
- `~/.sm/profiles/<profile>/<skill>/` stores skill content. `sm` changes
  visibility only; it does not run Git or synchronize repositories.

Do not confuse Pi extensions with Pi skills. Extensions execute TypeScript at
startup and can change runtime/UI behavior. Skills are instructions discovered
progressively and exposed by `sm` as symlinks.

## Distribution Decision

Default personal extensions to Home Manager, not npm:

1. Put the TypeScript source under `~/dotfiles/pi/extensions/`.
2. Add one `home.file` out-of-store symlink in `pi-agent.nix` when introducing a
   new extension.
3. Build all affected Home Manager outputs.
4. Activate only after the user approves the diff.

Use npm only when the user explicitly needs independent public distribution to
machines that do not consume these dotfiles. npm introduces package metadata,
versioning, tests, release authentication, registry availability, and update
maintenance. Publishing, deprecating, and unpublishing are external actions and
require explicit user intent.

Before implementing a key workaround, verify whether current Pi already
supports it and whether the terminal preserves the modifier. An extension
cannot distinguish two keys when the terminal emits identical bytes.

## Required Pi Documentation

Pi is installed under:

```text
/opt/homebrew/lib/node_modules/@earendil-works/pi-coding-agent
```

On Linux, resolve the installed package path instead of assuming the Homebrew
prefix. Read the relevant Markdown files completely and follow their linked
references before implementation:

- `docs/extensions.md`
- `docs/keybindings.md`
- `docs/tui.md` for custom editors or components
- `docs/terminal-setup.md` for modified-key problems
- `docs/packages.md` only for npm/git distribution
- `docs/skills.md` for skill discovery

Prefer Pi's existing extension APIs and examples. For editor input behavior,
wrap the previous editor factory when possible so extensions compose.

## Change Workflow

### 1. Establish Context

```sh
id -un
printf '%s\n' "$HOME"
git -C "$HOME/dotfiles" status --short --branch
pi --version
```

Read the target host, `flake.nix`, `modules/programs/pi-agent.nix`, the extension
source, and every relevant imported module. Preserve unrelated worktree changes.

Map the current Veno host conservatively:

- Darwin: `veno@macbook`
- Linux: `veno@thinkpad`

Never apply Veno's configuration to a Hermes account.

### 2. Implement Locally

For a new personal extension:

```text
~/dotfiles/pi/extensions/<name>.ts
~/.pi/agent/extensions/<name>.ts -> ~/dotfiles/pi/extensions/<name>.ts
```

Use an out-of-store Home Manager link so `/reload` sees source edits without a
new Nix build. A Home Manager switch is still required when the link declaration
is added, renamed, or removed.

Do not add a local personal extension to the `packages` array in
`pi/settings.json`. That array remains for packages distributed by npm or git.

### 3. Validate Before Activation

Run focused extension checks first. Use a temporary Pi config for startup/TUI
smoke tests when package state or credentials should be isolated. Verify actual
terminal bytes for keyboard fixes, ideally through the same tmux path used in
normal operation.

Show and inspect the complete proposed diff:

```sh
git -C "$HOME/dotfiles" diff --check
git -C "$HOME/dotfiles" diff -- modules/programs/pi-agent.nix pi/extensions pi/settings.json
git -C "$HOME/dotfiles" status --short
```

Because `pi-agent.nix` is shared, validate both Veno outputs. Build the current
host and at least evaluate the other host; report any platform limitation:

```sh
home-manager build --flake "path:$HOME/dotfiles#veno@macbook"
home-manager build --flake "path:$HOME/dotfiles#veno@thinkpad"
```

Use `path:` so untracked files are included during pre-commit validation.

### 4. Activate Only With Approval

`pi-sync` is the normal current-user activation path. It performs:

1. `git -C ~/dotfiles pull --ff-only`
2. `home-manager switch --flake path:...#<current Veno host>`
3. `pi update --extensions`

Run it only after a successful build, reviewed diff, and explicit activation
request. Never use `sudo`, `su`, or another user's credentials.

After activation, confirm the extension link and reload Pi:

```sh
readlink "$HOME/.pi/agent/extensions/<name>.ts"
```

Use `/reload` in an existing Pi session, or start a new session. Re-run the
behavioral smoke test after activation.

## sm Coordination

Pi discovers both `~/.agents/skills` and `~/.pi/agent/skills`. This workflow
skill lives at:

```text
~/.sm/profiles/pi-maintenance/pi-extension-home-manager/
```

Expose it through the `agents` target, which avoids unrelated legacy collisions
in the direct Pi target:

```sh
sm enable pi-maintenance --target agents
sm apply --target agents
sm status --target agents
```

For skills intentionally grouped under `pi-local`, inspect and operate the Pi
target explicitly because the configured default target is `agents`:

```sh
sm enabled --target pi
sm apply --target pi
```

`sm apply` reconciles links without changing profile precedence. `sm enable`
raises the named profile to highest precedence. Check for unmanaged collisions
before changing target contents. Do not manually replace links that `sm` owns.

If `~/.sm` is a Git checkout, inspect its status and synchronize it with normal
Git commands. If it is not a Git checkout, do not initialize or add a remote
without an explicit request.

## Completion Checklist

- Personal extension source lives under `~/dotfiles/pi/extensions/`.
- Home Manager owns only the extension link, not mutable Pi state or skills.
- No npm package was introduced unless public distribution was requested.
- Current-host build succeeded; every other affected host was validated or its
  limitation was reported.
- The user reviewed the diff before activation.
- `pi-sync` or `home-manager switch` ran only with explicit approval.
- The installed link resolves to the dotfiles source.
- `/reload` or restart loaded the extension.
- Keyboard/UI behavior was tested through the real terminal path.
- The appropriate `sm` target was reconciled and exposes the skill to Pi.
