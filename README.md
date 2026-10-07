# sbx

Docker Sandbox environment files for running coding agents against
**any** project on this machine, kept deliberately outside every project's
own repo.

## Requirements

- [Docker Sandboxes](https://docs.docker.com/ai/sandboxes/) — the `sbx`
  CLI, installed and signed in.
- [Doppler](https://www.doppler.com/) — the `doppler` CLI, installed and
  authenticated, with a project/config already holding a secret named
  `GH_TOKEN` (a GitHub token scoped to whatever access the sandboxed
  agent should have).

## Why this lives outside the project it sandboxes

A Docker Sandbox's `sbxenv.yaml` isn't just read by the agent — its
`secrets:`/`lifecycle:` entries are resolved and executed **on the host**,
with the real user's own privileges, not inside the sandbox. If these
files sat inside a project's own mounted directory (even read-only, even
in a dotfolder), an agent running in the sandbox could rename the
containing folder — an ordinary filesystem operation on an otherwise
writable mount — and drop a replacement file in its place. The next time
a human launches the sandbox from that project and approves the plan
without reading it closely, whatever the replacement file says would run
for real, on the host.

Keeping these files in one folder that's never mounted into any sandbox
removes that risk entirely: nothing running inside a sandbox can read,
rename, or write anything here, because nothing inside a sandbox has a
path to this folder at all. One folder, reused for every project.

## Not tied to one agent, or one project

This is sandbox infrastructure for working with coding agents in
general, not a Claude/Codex-specific thing, and nothing in it is
specific to any one project.

- The **workspace mounted into the sandbox is whichever project directory
  you run the launcher command from**. See "How it works" below.
- **Agent selection is a one-line overlay file per agent**, chosen by name
  at the command line (`aibox claude`, `aibox codex`) rather than a
  separate command per agent. Right now there are two overlays —
  `envs/sbxenv.claude.yaml` and `envs/sbxenv.codex.yaml` — because those
  are the agents in use today. Adding support for another agent (anything
  in `sbx create --help`'s "Available agents" list, or a custom kit)
  means adding one more `sbxenv.<agent>.yaml` file to `envs/` — nothing
  else here changes, and `aibox` picks it up immediately, no reinstall
  needed.
- **Which Doppler project/config to read from is passed in per call**,
  never hardcoded here. See Usage.

## Layout

```
sbx/
├── envs/
│   ├── sbxenv.yaml          # shared base
│   ├── sbxenv.claude.yaml   # agent: claude
│   └── sbxenv.codex.yaml    # agent: codex
└── install.sh
```

- `envs/sbxenv.yaml` — the shared base, parameterized via a required
  `args:` block: `workspace`, `doppler_project`, `doppler_config`. None
  of them have defaults; all three must be supplied on every call, or
  `sbx` refuses to run.

  It declares the sandboxed agent's GitHub token as a `github` service
  secret, resolved via `doppler secrets get GH_TOKEN --plain -p
  ${{ env.args.doppler_project }} -c ${{ env.args.doppler_config }}`.
  This runs on the host through the host's authenticated Doppler CLI
  session — the raw token value never enters the sandbox. Verified live:
  the sandbox only ever sees a proxy-managed sentinel value; the real
  credential is injected by `sbx`'s host-side proxy only on outbound
  requests to GitHub's own hosts.
- `envs/sbxenv.claude.yaml` / `envs/sbxenv.codex.yaml` — one-line overlays
  that pick the agent (`agent: claude` / `agent: codex`). `sbx env run`
  deep-merges whichever overlay `aibox` names after the base file, later
  file winning (docker-compose `-f` semantics).
- `install.sh` — (re)generates the `aibox` launcher below, pointed at an
  envs directory (default `envs/`, or a path passed as its argument — see
  Setup). Re-run it any time that directory moves or is cloned to a new
  path (it resolves the absolute path at run time and bakes it into the
  generated command). Adding a new `sbxenv.<agent>.yaml` file to that
  directory does **not** require re-running this — `aibox` looks up
  overlay files by name at call time, not at install time.

## How it works

`aibox <agent> --doppler-project <project> --doppler-config <config>`:
1. Checks `sbxenv.<agent>.yaml` exists in the envs directory baked in at
   install time; errors with the list of agents that actually have an
   overlay file if not.
2. Reads the directory it was **called from** (`$PWD`) and passes it as
   `--env-arg workspace=$PWD`, so the sandbox mounts whatever project
   you're currently standing in — the envs directory's own location
   never matters for that part.
3. Requires `--doppler-project`/`--doppler-config` and errors immediately
   if either is missing — there is no default, by design, so this repo's
   history never records which project/config any particular caller
   happened to use.
4. Runs `sbx env run <envs-dir>/sbxenv.yaml <envs-dir>/sbxenv.<agent>.yaml`
   with those args.

## Setup

```
./install.sh                # uses envs/ next to this script
./install.sh /path/to/envs  # or point it at a different directory
```

Installs `aibox` into `~/.local/bin` (already expected to be on `PATH`).
Errors immediately if the given directory (or the default `envs/`)
doesn't exist, or doesn't contain an `sbxenv.yaml`.

## Usage

From inside any project directory:

```
aibox claude --doppler-project <project> --doppler-config <config>
aibox codex  --doppler-project <project> --doppler-config <config>
```

Mounts `$PWD` as the sandbox's workspace and fetches the GitHub token
from the given Doppler project/config. Omitting the agent, naming one
with no matching `sbxenv.<agent>.yaml`, or omitting either Doppler flag
all fail fast with a usage error instead of silently falling back to
anything.

## Subscription auth and skills

Claude/Codex subscription login (`sbx secret set anthropic --oauth` /
`sbx secret set openai --oauth`) and the global skills store
(`sbx skills import`) are separate, per-machine `sbx` state — not
declared here, not tied to this folder, shared across every sandbox on
this machine regardless of which project it's for. See `sbx secret ls`
and `sbx skills ls` to check what's currently configured.
