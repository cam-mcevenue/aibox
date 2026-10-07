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
- **Agent selection is just a one-line overlay file per agent.** Right now
  there are two — `sbxenv.claude.yaml` and `sbxenv.codex.yaml` — because
  those are the agents in use today. Adding support for another agent
  (anything in `sbx create --help`'s "Available agents" list, or a custom
  kit) means adding one more `sbxenv.<agent>.yaml` file and one more
  `agent: <name>` case in `install.sh` — nothing else here changes.
- **Which Doppler project/config to read from is passed in per call**,
  never hardcoded here. See Usage.

## Files

- `sbxenv.yaml` — the shared base, parameterized via a required `args:`
  block: `workspace`, `doppler_project`, `doppler_config`. None of them
  have defaults; all three must be supplied on every call, or `sbx`
  refuses to run.

  It declares the sandboxed agent's GitHub token as a `github` service
  secret, resolved via `doppler secrets get GH_TOKEN --plain -p
  ${{ env.args.doppler_project }} -c ${{ env.args.doppler_config }}`.
  This runs on the host through the host's authenticated Doppler CLI
  session — the raw token value never enters the sandbox. Verified live:
  the sandbox only ever sees a proxy-managed sentinel value; the real
  credential is injected by `sbx`'s host-side proxy only on outbound
  requests to GitHub's own hosts.
- `sbxenv.claude.yaml` / `sbxenv.codex.yaml` — one-line overlays that pick
  the agent. `sbx env run` deep-merges whichever overlay follows the base
  file, later file winning (docker-compose `-f` semantics).
- `install.sh` — (re)generates the launcher commands below, one per
  `sbxenv.<agent>.yaml` file present. Re-run it any time this folder
  moves or is cloned to a new path (it resolves its own absolute location
  at run time and bakes that path into the generated commands), and any
  time you add a new agent overlay file.

## How it works

Each generated launcher command:
1. Reads the directory it was **called from** (`$PWD`) and passes it as
   `--env-arg workspace=$PWD`, so the sandbox mounts whatever project
   you're currently standing in — this folder's own location never
   matters for that part.
2. Requires `--doppler-project <project>` and `--doppler-config <config>`
   on the command line and errors immediately if either is missing —
   there is no default, by design, so this repo's history never records
   which project/config any particular caller happened to use.
3. Runs `sbx env run <this-folder>/sbxenv.yaml <this-folder>/sbxenv.<agent>.yaml`
   with those args — the env files themselves always live here,
   regardless of which project is being sandboxed.

## Setup

```
./install.sh
```

Installs `sbx-claude` and `sbx-codex` into `~/.local/bin` (already
expected to be on `PATH`).

## Usage

From inside any project directory:

```
sbx-claude --doppler-project <project> --doppler-config <config>
sbx-codex  --doppler-project <project> --doppler-config <config>
```

Mounts `$PWD` as the sandbox's workspace and fetches the GitHub token
from the given Doppler project/config. Omitting either flag fails fast
with a usage error instead of silently falling back to anything.

## Subscription auth and skills

Claude/Codex subscription login (`sbx secret set anthropic --oauth` /
`sbx secret set openai --oauth`) and the global skills store
(`sbx skills import`) are separate, per-machine `sbx` state — not
declared here, not tied to this folder, shared across every sandbox on
this machine regardless of which project it's for. See `sbx secret ls`
and `sbx skills ls` to check what's currently configured.
