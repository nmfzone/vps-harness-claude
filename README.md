# Private Claude Code + ruflo + Hermes + Paseo VPS harness

Standalone Ansible project for Ubuntu 24.04+ that provisions a private VPS running
**Claude Code** as the coding agent, **Paseo** as the remote control plane,
**ruflo** for multi-agent orchestration, and **Hermes Agent** for persistent
memory, skills, a messaging gateway and scheduled automations. Paseo Desktop and
mobile connect over the private Tailscale network.

```text
Paseo Desktop/Mobile -- Tailscale --> VPS:6767 (UFW tailnet-only)
                                      └-> paseo.service (daemon + web UI)
                                          └-> Claude Code sessions (spawned by the daemon)
                                              └-> ruflo orchestration

Telegram / Discord -- outbound --> hermes.service (gateway + cron + persistent memory)
```

## How the pieces fit

- **Claude Code** (`@anthropic-ai/claude-code`) is the agent. Paseo launches it
  per session; there is no `claude.service`.
- **Paseo** (`@getpaseo/cli`) runs `paseo daemon run` under systemd, exposing the
  daemon API and its bundled web UI on `6767`. It is the replacement for a
  browser-based agent console.
- **ruflo** (`ruflo`) adds swarms, specialized agents and memory. It is
  initialized once in the shared workspace.
- **Hermes Agent** runs under its own `hermes.service` as a separate, always-on
  agent. It is *not* driven by Paseo.
- Model access for Claude Code and Hermes goes through the **OmniRoute gateway**.

The playbook explicitly registers three public marketplaces and installs
`andrej-karpathy-skills@karpathy-skills`, `agent-skills@addy-agent-skills`, and
`superpowers@claude-plugins-official` as the `agent` account. It verifies all
three are enabled before starting Paseo. The user-level Claude settings also
declare them in `enabledPlugins` and list their sources in
`extraKnownMarketplaces`. Installation requires outbound HTTPS access to GitHub
(and to a package registry if a plugin needs dependencies); no GitHub account
is required for these public repositories.

Paseo is intentionally unauthenticated only over the tailnet; set
`PASEO_PASSWORD` (a random value is generated if you omit `vault_paseo_password`)
and rely on the UFW `tailscale0`-only rule on port 6767.

## Prerequisites

On the operator workstation:

- Ansible Core plus the `ansible.posix` and `community.general` collections.
- SSH access to an Ubuntu 24.04 VPS using a bootstrap account with `sudo`.
- An SSH key for the bootstrap account.
- A Tailscale tailnet and an auth key suitable for this node.
- Paseo Desktop (or the Paseo mobile app) installed.

Upstream references:

- Paseo: <https://paseo.sh/docs>
- Paseo connectivity (SSH / relay / Tailscale): <https://paseo.sh/docs/connectivity>
- Claude Code: <https://docs.anthropic.com/en/docs/claude-code>
- ruflo: <https://github.com/ruvnet/ruflo>
- Hermes Agent: <https://hermes-agent.nousresearch.com/docs/>
- Tailscale Linux install: <https://tailscale.com/download/linux>

## Operator configuration

Create the ignored vault file and fill in the target address, administrator
credentials, SSH key, Tailscale auth key, OmniRoute token, existing Telegram
BotFather token, and your numeric Telegram user ID:

```bash
cp group_vars/vps/vault.yml.example group_vars/vps/vault.yml
$EDITOR group_vars/vps/vault.yml
ansible-vault encrypt group_vars/vps/vault.yml
```

`vault_omniroute_token` is used both as `ANTHROPIC_AUTH_TOKEN` for Claude Code and
as the OpenAI-compatible key for Hermes. Secrets never appear in a tracked file:
agent configuration is rendered from `roles/agent_config/templates/*.j2` at deploy
time (the `*.example` files document the shape).

## Go toolchain

The playbook installs **Go 1.27.1** from a SHA-256-verified official Linux archive
for the VPS architecture (`x86_64` or `aarch64`). It puts Go in `/usr/local/go`
and links `go` and `gofmt` under `/usr/local/bin`, which is on both the admin
and agent users' normal PATH and on `paseo.service`'s PATH. The playbook checks
`go version` as both accounts. A different Go version under `/usr/local/go` is
replaced; Go installations managed elsewhere are left alone.

## Optional Git SSH access for the agent

Set `vault_agent_git_ssh_hosts` in the encrypted
`group_vars/vps/vault.yml` to the SSH Git hosts you intend the agent to use.
See the commented example in `vault.yml.example`. If omitted, the default
empty list does nothing. For example, inside the encrypted vault:

```yaml
vault_agent_git_ssh_hosts:
  - alias: abc-gitlab
    hostname: gitlab.com
    port: 31022
    identity: id_abc_ed25519
```

Ansible generates each agent SSH key pair on the VPS once, scans the listed
server's ed25519 host key into `/var/lib/agent/.ssh/known_hosts`, and configures
the matching identity for both the host alias and hostname. It **does not clone**
any repository or grant access on the Git server. Retrieve the public key after
deployment and register it with the Git server as a read-only deploy key:

```bash
sudo cat /var/lib/agent/.ssh/id_abc_ed25519.pub
```

This deliberately trusts whichever host key answers `ssh-keyscan` during
provisioning (trust on first use), so it is not protected against an active
impersonator at that moment. Validate the resulting host fingerprint against
one from your Git administrator when possible. After the public key is granted,
Claude can clone an SSH repository URL without a first-host-key prompt.

## Deploy

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml --ask-vault-pass --check   # dry run
ansible-playbook site.yml --ask-vault-pass           # apply
```

The playbook installs Node.js 22 and the four agent CLIs via npm, registers the
hosted grep.app HTTP MCP server for Claude Code, and starts `paseo.service` and
`hermes.service`. It runs `hermes pm install --extra messaging` before starting
the Hermes gateway. It does not clone or build the grep.app MCP server locally.

## Hermes configuration

Two files carry Hermes' settings, both owned by the `hermes` role and rendered
from vault at deploy time:

- `~/.hermes/.env` — provider credentials (`OPENAI_API_KEY`,
  `OPENAI_BASE_URL`) and your existing bot's `TELEGRAM_BOT_TOKEN` and
  `TELEGRAM_ALLOWED_USERS`, rendered from Ansible Vault with mode `0600`.
  Hermes uses Telegram long polling by default, so no inbound webhook port
  is needed. Stop any other process using that bot token before deploying;
  two pollers for one bot will conflict.
- `~/.hermes/config.yaml` — **provider, base URL and model**, applied with
  `hermes config set` (set by `hermes_model_config` in `group_vars/vps/vars.yml`).
  `hermes config set` merges into the installer-managed file, preserving its
  `_config_version` migration marker, so it is never templated wholesale.

Ansible manages the system-level `hermes.service`, which runs `hermes gateway run
--external-supervisor` and restarts through systemd. Do **not** also run `hermes
gateway install` or `hermes gateway start`: those would create a second service.
The Telegram variables connect the existing bot but do not create a new bot or
set up a home channel for cron delivery. For Telegram-delivered scheduled
results, configure `TELEGRAM_HOME_CHANNEL` separately.

Verify the wiring, and adjust the model if needed:

```bash
sudo -u agent -i
hermes doctor
hermes model            # interactive picker, if you want a non-default model
```

If `hermes doctor` reports the key is missing, the "custom" provider may not read
`OPENAI_API_KEY` from `.env` in your build; set it explicitly (this writes the
token into `config.yaml`):

```bash
hermes config set model.api_key "$OPENAI_API_KEY"
```

## Routing coding work through Paseo (Hermes skill)

The `hermes` role installs a `paseo-coding` skill at
`~/.hermes/skills/paseo-coding/SKILL.md`, adds it to
`skills.auto_load` in Hermes' `config.yaml`, and adds `PASEO_PASSWORD` to Hermes'
`.env`. New CLI, Telegram, cron, and API sessions load the skill in the initial
prompt. The skill instructs Hermes to find one local Paseo workspace named
**Coding Workspace** for `<workspace_root>`, or create it if missing.
It serializes the find-or-create operation with `flock` to prevent simultaneous
requests from creating duplicates, then uses the complete workspace ID and a
task-specific agent title on every coding request:

```bash
paseo run --provider claude --workspace "<workspaceId>" --cwd <workspace_root> \
  --title "<short task title>" --background --format json "<task>"
```

This reuses one workspace rather than creating another one per run. Existing
duplicate workspaces are left untouched. If more than one workspace matches,
Hermes stops and reports the ambiguity instead of guessing. The coding agent
remains inspectable by Paseo, then Hermes reports the result on Telegram.
Hermes keeps its own tools for non-coding work.

This is **advisory**: a skill guides the model, it does not remove Hermes' file or
terminal tools. To force the behavior you would instead restrict Hermes' Telegram
toolsets (`platform_toolsets.telegram`), which this harness does not do.

Verify after deploy:

```bash
sudo -u agent env HOME=/var/lib/agent HERMES_HOME=/var/lib/agent/.hermes \
  /var/lib/agent/.hermes/hermes-agent/.hermes/bin/hermes skills list
sudo -u agent env HOME=/var/lib/agent \
  paseo workspace ls --json    # find "Coding Workspace" and copy its workspaceId
sudo -u agent env HOME=/var/lib/agent \
  paseo run --provider claude --workspace "<workspaceId>" \
  --cwd /home/<admin>/Workspace --title "Verify shared workspace" "echo hello"
```

## ruflo initialization

`ruflo init` is a wizard. The playbook runs it once, non-interactively, guarded by
`~/.claude-flow`; if the installed CLI has no non-interactive mode the task fails
on first run. In that case initialize it by hand and re-run the playbook:

```bash
sudo -u agent -i
cd ~/Workspace && ruflo init
```

Note: Paseo spawns Claude Code through the Claude Agent SDK, so verify that any
ruflo hooks fire in Paseo-launched sessions; otherwise ruflo applies to
interactive SSH sessions.

## CodeGraph indexing

The playbook registers CodeGraph as a user-scoped Claude Code MCP server for the
`agent` account (`/var/lib/agent/.claude.json`). It also marks Claude Code's
onboarding complete for this headless gateway account. That flag is undocumented
Claude Code state; authentication still depends on `ANTHROPIC_AUTH_TOKEN` in the
user settings, not on the onboarding flag. New Claude Code sessions, including
those launched by Paseo, can use `codegraph_explore`. Restart an existing session
to load the server; check its connection with `claude mcp list` or `/mcp`.

Ansible runs `codegraph init --yes {{ workspace_root }}` once as the `agent`
account, creating an initial Workspace index. An empty Workspace initially
indexes no source files. This is not a guarantee that repositories cloned in
the future will automatically be indexed: initialize each new repository as
the `agent` account for reliable repo-specific results:

```bash
cd /home/<admin>/Workspace/<repo>
sudo -u agent env HOME=/var/lib/agent /var/lib/agent/.local/bin/codegraph init
sudo -u agent env HOME=/var/lib/agent /var/lib/agent/.local/bin/codegraph status
```

The `agent` account has a non-login shell, so run the installed binary via
`sudo -u agent` rather than `sudo -u agent -i`.

## Connect Paseo

Open the Paseo web UI at the VPS Tailscale address and port `6767`, for example
`http://vps-harness-claude:6767`, then enter `PASEO_PASSWORD`. In Paseo Desktop,
add the daemon as a host (direct/Tailscale, or SSH). The daemon is already running
in the background.

## Use the VPS as a Tailscale exit node

The `tailscale_advertise_exit_node` setting in `group_vars/vps/vars.yml` advertises
the VPS as an exit node (and enables persistent IPv4/IPv6 forwarding) when true.
Advertisement alone does **not** route traffic: approve **Use as exit node** in the
Tailscale admin console, then select it on the client. See
<https://tailscale.com/kb/1103/exit-nodes>. Confirm egress and DNS before relying on
it; keep SSH and the tailnet-only Paseo rule intact.

## Operations

```bash
sudo systemctl status paseo.service hermes.service
sudo journalctl -u paseo.service -f
sudo journalctl -u hermes.service -f
curl -I http://vps-harness-claude:6767/   # from a tailnet workstation
```

Do not add a public inbound firewall rule for the agent services. Paseo is reachable
only over the tailnet; Hermes' gateway is outbound-only.
