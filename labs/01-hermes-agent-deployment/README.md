# Lab 01: Hermes Agent Deployment

> Part of [hermes-agent-rea-lab](../../README.md). Next: [Lab 02: REA Integration](../02-rea-integration/README.md)

**Date:** November 2026
**Status:** Complete. Hermes Agent runs in a dedicated Ubuntu Server VM on Proxmox and is reachable through Slack and Hermes Desktop. Slack model switching is handled with a per-channel override (see [Slack model switching](#slack-model-switching)).

## Overview

This lab deploys [Hermes Agent](https://hermes-agent.nousresearch.com/), Nous Research's open-source personal agent, as a self-hosted service in my Proxmox homelab. Hermes keeps persistent memory, runs tools and shell commands on its host, and connects to chat platforms, so I can hand it tasks from Slack or the Hermes Desktop app.

The agent gets its own VM: it has shell access and holds API credentials, so it's isolated from the rest of the homelab and can be snapshotted and rolled back as a unit.

## Environment

| Component | Details |
|---|---|
| Proxmox host | Intel Core i7-4790, 32 GB DDR3, Radeon Pro WX 5100 (8 GB) |
| Hypervisor | Proxmox VE 9.2.2 |
| VM | `clark-hermes`: 2 cores (CPU type `host`), 4 GB RAM, 32 GB disk on ZFS |
| OS | Ubuntu Server 26.04.1 LTS, kernel 7.0.0-42-generic |
| Agent | Hermes Agent v0.22.5 |
| Model providers | OpenRouter (default: `thinkingmachines/inkling:free`); OpenAI Codex via ChatGPT Plus |
| Access | Slack (Socket Mode) and Hermes Desktop (via the Hermes dashboard on the LAN) |

### Why hosted models instead of local inference

I considered running the model on the host's GPU. Hermes needs a model with a large context window (64K) and reliable tool calling, and the host's 8 GB Polaris-generation card has no ROCm support. Running a capable model locally wasn't practical, so inference goes through hosted providers and the VM only runs the agent itself, which needs little CPU or RAM.

## Architecture

```
Slack (DM / #hermes-home)          Hermes Desktop (on my PC)
        │ Socket Mode (outbound)           │ HTTP, LAN only, password login
        ▼                                  ▼
hermes-gateway.service           hermes-dashboard.service (port 9119)
        └──────────────┬───────────────────┘
                       ▼
        Hermes Agent — VM clark-hermes (Proxmox)
                       │
                       ├──▶ OpenRouter ──▶ Inkling (default)
                       └──▶ OpenAI Codex (ChatGPT Plus, OAuth)
```

Hermes runs as two systemd **user** services under my account, not root:

| Service | Role | Created by |
|---|---|---|
| `hermes-gateway.service` | Connects Hermes to Slack | Hermes setup wizard |
| `hermes-dashboard.service` | Web dashboard that Hermes Desktop connects to | Me (custom user unit) |

## Procedure

### 1. Create the VM

1. Downloaded the Ubuntu Server LTS (live server, amd64) ISO in Proxmox under **local (pve01) → ISO Images → Download from URL**.
2. Created the VM: 2 cores with CPU type `host`, 4 GB RAM, and a 32 GB disk on the ZFS pool, with a VirtIO network adapter on the LAN bridge.
3. Installed Ubuntu Server with the default LVM storage layout, OpenSSH server enabled, no snaps, hostname `clark-hermes`, and a static IP on the home LAN.

All later work was done over SSH.

### 2. Install Hermes Agent

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

The installer puts Hermes in `~/.hermes/hermes-agent` and the `hermes` command in `~/.local/bin/`. Then I ran the setup wizard:

```bash
hermes setup
```

In the wizard I:

- Chose **OpenRouter** as the provider and entered my API key.
- Selected `thinkingmachines/inkling:free` as the default model.
- Chose **Slack** as the messaging platform and said **yes** to installing the gateway service.
- Disabled tools I don't need on this VM: image generation, text-to-speech, and computer use.

### 3. Connect Slack

Hermes generates a ready-made Slack app manifest:

```bash
hermes slack manifest --write
```

1. At api.slack.com/apps, chose **Create New App → From an app manifest** and pasted the generated manifest. The app uses **Socket Mode**, so Slack never needs to reach into my network.
2. Installed the app to my workspace to get the bot token.
3. Under **Basic Information → App-Level Tokens**, generated an app-level token with the `connections:write` scope (required for Socket Mode).
4. Entered both tokens in the Hermes wizard, plus my Slack member ID in the **allowed users** field so only I can command the agent.

The gateway is managed with:

```bash
hermes gateway status
hermes gateway restart
```

### 4. Add OpenAI Codex as a second provider

To use my ChatGPT Plus subscription alongside OpenRouter, I signed in to the Codex provider over OAuth (the `--no-browser` flag prints a link to open on another device, since the VM is headless):

```bash
hermes auth add openai-codex --type oauth --no-browser
hermes auth status openai-codex     # openai-codex: logged in
```

Then I tested it with a one-shot prompt:

```bash
hermes --provider openai-codex -m gpt-6-luna -z "Reply with only the word PONG."
# PONG
```

### 5. Connect Hermes Desktop through the dashboard

Hermes Desktop connects to the VM's Hermes **dashboard** over the LAN, not over SSH or Slack. The dashboard needs a login first.

**Create the dashboard login.** The password hash must be generated with Hermes's own Python environment:

```bash
cd ~/.hermes/hermes-agent
source ./activate
HASH=$(python3 -c 'from getpass import getpass; from plugins.dashboard_auth.basic import hash_password; print(hash_password(getpass("Dashboard password: ")))')
SECRET=$(openssl rand -base64 32)
```

I checked that `$HASH` wasn't empty, then added three values to `~/.hermes/.env` (values not shown):

```
HERMES_DASHBOARD_BASIC_AUTH_USERNAME=...
HERMES_DASHBOARD_BASIC_AUTH_PASSWORD_HASH=...
HERMES_DASHBOARD_BASIC_AUTH_SECRET=...
```

and locked the file down:

```bash
chmod 600 ~/.hermes/.env
```

**Run the dashboard as a service.** I created a systemd user unit at `~/.config/systemd/user/hermes-dashboard.service`:

```ini
[Unit]
Description=Hermes dashboard

[Service]
ExecStart=%h/.local/bin/hermes dashboard --no-open --host <VM-IP> --port 9119
Restart=on-failure

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now hermes-dashboard
systemctl --user status hermes-dashboard     # active (running)
```

The dashboard binds to the VM's LAN address only, and the firewall allows port 9119 from the home LAN only (see [Hardening](#6-hardening)).

**Connect the app.** In Hermes Desktop: **Settings → Gateway → Remote Gateway**, entered `http://<VM-IP>:9119`, signed in with the dashboard login, saved, and reopened the app.

### 6. Hardening

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent

# Firewall: SSH and dashboard reachable from the home LAN only
sudo ufw allow from <home-LAN-subnet> to any port 22 proto tcp
sudo ufw allow from <home-LAN-subnet> to any port 9119 proto tcp
sudo ufw enable

# Keep user services running without an active login session
sudo loginctl enable-linger $USER
```

### 7. Snapshot

Took a Proxmox snapshot, `hermes-working`, once everything was running.

### 8. Verify

- **Slack:** Hermes replied in a DM and ran real commands when asked about the OS and disk space.
- **Hermes Desktop:** Asked "What is this machine's hostname?" and got back `clark-hermes`.
- **Codex provider:** The PONG test in step 4 returned `PONG`.
- **Services:** `hermes-gateway` and `hermes-dashboard` both active, and both came back on their own after a VM reboot.

## Problems and fixes

| # | Problem | Cause | Fix |
|---|---|---|---|
| 1 | No **ISO Images** option in Proxmox | I'd selected the node `pve01`, not its storage | Expanded `pve01` and selected **local (pve01) → ISO Images** |
| 2 | The planned free model wasn't available | `z-ai/glm-5.2:free` had left OpenRouter's free tier | Checked the current free list and switched to `thinkingmachines/inkling:free` |
| 3 | No CLI-only option on the wizard's messaging screen | The wizard expects a platform | Chose Slack, which I wanted anyway |
| 4 | Hermes reported the disk as 15 GB with 4.8 GB free | Hermes ran `df -h` without a path and misread the output | Checked it myself: `sudo vgs`, `sudo lvs`, and `df -h /` all showed ~30 GB with 19 GB free. No fix needed, but a good reminder to verify what the agent reports |
| 5 | Codex not listed in `hermes model` | That screen was OpenRouter's model list, not the provider list | Chose **Skip (keep current)** and added Codex with `hermes auth add openai-codex` |
| 6 | `bash: gpt-6-luna: No such file or directory` | I kept the `< >` from a placeholder, and bash read `<` as input redirection | Removed the angle brackets |
| 7 | Dashboard password step failed with `venv/bin/python3: No such file or directory`, and the rest of the block wrote an empty hash to `.env` | Hermes doesn't use a standard `venv`; its Python lives in Hermes-managed environments under `~/.hermes/tools/` | Removed the bad lines (`sed -i '/^HERMES_DASHBOARD_BASIC_AUTH_/d' ~/.hermes/.env`), ran `source ./activate` to use Hermes's Python, and reran with a check that the hash isn't empty before writing it |
| 8 | `cua-driver` error about missing `libXi.so.6` during `source ./activate` | That driver is for the Computer Use tool | Ignored, since Computer Use is disabled on this headless VM |

## Slack model switching

**Goal:** Keep Inkling (free) as the default, and use Codex (ChatGPT Plus) when I need it.

**Problem:** Switching with `/model` in Slack never stuck. Slack confirmed "Model switched," but replies still ran on Inkling.

**Investigation:**

1. Asking the agent which model it was on wasn't reliable, so I used the logs as the source of truth. `agent.log` contains a binary character, so `grep` needs `-a` to search all of it:
   ```bash
   grep -a -n "provider=" ~/.hermes/logs/agent.log | tail -8
   ```
2. Matching session IDs between `gateway.log` and `agent.log` showed the cause. My reply in the `/model` thread ran in a **new session** (`history=0`) on Inkling. Hermes treats each Slack thread as its own session, and the session ID ends in the thread's timestamp.
3. Slack doesn't allow app slash commands inside threads ("/model is not supported in threads"). In a channel, `/model` runs at the top level, so the override attaches to a session that thread replies never use. In the Hermes DM, every conversation is a thread, so `/model` can't run at all.
4. `/new` and `hermes gateway restart` made it worse, since both reset session-only overrides back to the default.
5. Some Inkling traffic is expected regardless: the `auxiliary` jobs (titles, compression, session search) are set to `auto` and use the default provider.

**Fix: a dedicated Codex channel.** Hermes supports per-channel model overrides, keyed by Slack channel ID. I created `#hermes-codex` and added this under `platforms: → slack:` in `~/.hermes/config.yaml` (backup: `config.yaml.pre-channel-override`):

```yaml
platforms:
  slack:
    channel_overrides:
      "<CHANNEL_ID>":
        model: gpt-6-luna
        provider: openai-codex
```

The gateway reads overrides at startup, so it needed a restart:

```bash
hermes gateway restart
```

**Verification:** After the restart, a message in `#hermes-codex` logged `model=gpt-6-luna provider=openai-codex`, while Hermes Desktop and `#hermes-home` stayed on Inkling.

| Where I post | Model |
|---|---|
| `#hermes-home`, DM, Hermes Desktop | Inkling via OpenRouter (default) |
| `#hermes-codex` | `gpt-6-luna` via Codex (ChatGPT Plus) |

Switching models now just means posting in a different channel.

## Security considerations

- **Dedicated sandbox:** The VM holds no Proxmox credentials and no SSH keys to other machines. If the agent is ever misused, the damage stays in one snapshot-able VM.
- **No inbound exposure:** Slack uses Socket Mode (an outbound connection), so no ports are forwarded on the router.
- **Restricted access:** Only my Slack user ID can command the agent. SSH and the dashboard are reachable only from the home LAN.
- **Credentials:** API keys, Slack tokens, and the dashboard secrets live in `~/.hermes/.env` with `600` permissions, and none of them are in this repo.
- **Plain HTTP on the LAN:** The dashboard is password-protected but not encrypted. That's acceptable on my home LAN, but HTTPS is on the next-steps list.
- **Not root:** Hermes runs as a regular user via systemd user services.
- **Data sharing:** Everything sent to the agent goes to the model provider and is subject to its data policy.

## Lessons learned

1. **Verify what the agent tells you.** Hermes misreported the disk size and then offered an unconvincing explanation. Checking with `vgs`, `lvs`, and `df -h /` settled it in seconds.
2. **Free models come and go.** The model I planned on had left the free tier. Check the provider's current list before relying on one.
3. **Know which screen you're on.** The "missing" Codex provider was only missing from a model list, not the provider list.
4. **Placeholders are a trap in bash.** `<` and `>` are redirection operators, so leftover angle brackets cause confusing errors.
5. **Check values before you write them.** A failed command inside a block still let the rest run and save an empty password hash. Testing that the value isn't empty first prevents that.
6. **Small VM, hosted models.** With inference hosted, 2 cores and 4 GB of RAM run Hermes comfortably.
7. **Logs beat self-reports.** The model's claims about its own model were sometimes right and sometimes wrong. Matching session IDs across `gateway.log` and `agent.log` gave a definitive answer.
8. **Know your platform's limits.** The routing "bug" was really Slack's thread model meeting Hermes's per-thread sessions. Moving the setting into config, rather than a chat command, avoided it entirely.

## Next steps

- [x] Extend Hermes with reverse-engineering tools: [Lab 02: REA Integration](../02-rea-integration/README.md)
- [x] Resolve the Slack model-routing issue (per-channel override)
- [x] Set a spending limit on the OpenRouter API key
- [ ] Tailscale for secure remote access to Hermes Desktop outside the home
- [ ] HTTPS for the dashboard through Nginx Proxy Manager
- [ ] Weekly Proxmox backup job for the VM
- [ ] Self-host SearXNG for the agent's web search

## References

- Hermes Agent: https://hermes-agent.nousresearch.com/
- OpenRouter: https://openrouter.ai/
- Slack apps and Socket Mode: https://api.slack.com/apis/socket-mode
- Proxmox VE: https://www.proxmox.com/en/proxmox-virtual-environment
