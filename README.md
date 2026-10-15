# pi-sandbox

Personal setup for running the [Pi coding agent](https://www.npmjs.com/package/@earendil-works/pi-coding-agent)
in a rootless podman container, with custom skills and prompts.
The agent can read the project (`$PWD` mounted at `/workspace`) and the mounted gcloud ADC file.

## Layout

| Path | What |
|---|---|
| `pi-sandbox` | Launcher script (install as `~/.local/bin/pi-sandbox`) |
| `Dockerfile.pi` | Image `pi-sandbox`: openSUSE Tumbleweed + Node, ripgrep, jq, mkmedia, xorriso, bubblewrap, socat, SUSE CA |
| `config/pi/settings.json`, `config/pi/models.json` | Pi core config, mounted read-only (default model, Vertex/Ollama providers, packages incl. the `openqa` skill) |
| `config/extensions/permission-mode.json` | [pi-permission-modes](https://www.npmjs.com/package/pi-permission-modes) overrides (modes, network allowlist), layered on its defaults |
| `config/extensions/web-search.json`, `config/extensions/i-have-adhd.json` | Per-extension config for pi-web-access and i-have-adhd |
| `prompts/` | Prompt templates (`refactor`) |
| `skills-custom/` | Own skills: `download-iso`, `modify-iso`, `prepare-iso` (SLES 16.x test ISOs) |
| `extensions-custom/` | Custom extensions |
| `.env` | Non-sensitive config, gitignored; copy from `.env.example` |

## Usage

```bash
pi-sandbox [args]               # run the agent in a fresh --rm container; args go to pi (e.g. --perm plan)
pi-sandbox update --extensions  # update all packages in config/pi/settings.json (extensions and skills like openqa)
```

Third-party skills (`openqa`) come as pi packages in `config/pi/settings.json`, like extensions.
Projects are trusted by `defaultProjectTrust: "always"` (loads their `.pi/` resources without asking).

Permission modes (Default, Plan Mode, Build, YOLO; `alt+m` or `/perm <mode>`) come from
pi-permission-modes: allow/ask/deny policy plus a bubblewrap sandbox for bash.

All own config is mounted read-only. To change skills, prompts or `config/extensions/permission-mode.json`,
run a second `pi-sandbox` from `~/Code/pi-sandbox`, then `/reload` in the first one.
Directories update live; single-file mounts (`*.json`) may need a restart
if the file was replaced rather than edited in place. "Allow forever" can't persist from a
read-only mount (it lasts the session); add the domain to `config/extensions/permission-mode.json` instead.

Non-sensitive config (project ID, region) goes in `.env` (see `.env.example`).

## Rebuilding the agent image

```bash
cd ~/Code/pi-sandbox
podman build -f Dockerfile.pi -t pi-sandbox .
# fresh Pi version / base image (the npm layer is otherwise cached):
podman build --no-cache --pull=newer -f Dockerfile.pi -t pi-sandbox .
```

No restart needed: each `pi-sandbox` run starts a new container from the image.
State lives in the `pi-agent-home` volume and survives rebuilds.

To start from scratch (drops auth, sessions and installed packages; packages reinstall on the next run):

```bash
podman volume rm pi-agent-home  # no pi-sandbox container may be running
```

## Sandbox

- Rootless podman, `--cap-drop=ALL`, `no-new-privileges`.
- `--cap-add=SETFCAP` and `unmask=/proc/*` so bubblewrap can run inside the container (uid map + fresh `/proc`).
- `gh` gets `GH_TOKEN` from `~/.config/pi-sandbox/gh-token` (if present): a fine-grained PAT with public-repo read-only access, so the container can't write to GitHub; post reviews from the host.
- Only `application_default_credentials.json` is mounted (read-only), not the whole `~/.config/gcloud`; Pi's own `~/.pi/agent` is not mounted from the host.

## Todo

- [ ] Be able to run `/share` inside Pi and avoid error `Error: GitHub CLI is not logged in. Run 'gh auth login' first.`

