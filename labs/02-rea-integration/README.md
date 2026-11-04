# Lab 02: REA Integration — Adding Reverse-Engineering Tools to Hermes Agent

> Part of [hermes-agent-rea-lab](../../README.md). Previous: [Lab 01: Hermes Agent Deployment](../01-hermes-agent-deployment/README.md)

**Date:** November 1, 2026
**Status:** In progress. REA is connected to Hermes as an MCP server, trimmed to 26 static-analysis tools, and verified with a live tool call. The package metadata comparison for the SBOM exercise is still open.

## Overview

This lab connects [REA (Reverse Engineer Anything)](https://github.com/morluto/rea) to my self-hosted [Hermes Agent](https://hermes-agent.nousresearch.com/) so Hermes can inspect binaries, applications, and data files on request from Hermes Desktop or Slack.

REA is an open-source MCP (Model Context Protocol) server that gives AI agents reverse-engineering tools: binary layout and signature inspection, dependency extraction from Go and Rust binaries, JavaScript/Electron app analysis, .NET metadata, network capture parsing, and more. Hermes is Nous Research's open-source personal agent, which can load MCP servers and call their tools.

The goal is to build a lab for **software supply-chain and vendor-risk analysis**, the kind of work done in GRC and security compliance roles:

- What does this application contain, and what does it depend on?
- Is this executable properly signed, and by whom?
- What does this app send over the network?

## Environment

| Component | Details |
|---|---|
| Hypervisor | Proxmox VE 9.2.2 |
| Agent VM | Ubuntu Server 26.04 (headless) |
| Agent | Hermes Agent, accessed via Hermes Desktop and Slack |
| Node.js | 26.12.1 (bundled with Hermes at `~/.hermes/tools/node-26.12.1-linux-x64/`) |
| REA | 6.8.0 (`rea-agents` npm package) |
| Native analysis engines | None installed (Hopper, Ghidra, IDA, JADX, and Kuna are optional) |

## Architecture

```
Me (Hermes Desktop / Slack)
        │
        ▼
Hermes Agent  ──(LLM provider)──▶  model
        │
        │ MCP over stdio (local subprocess, no network port)
        ▼
REA MCP server  ──▶  reads local files on the VM, returns findings + evidence
```

REA runs as a local subprocess that Hermes launches and talks to over standard input/output. It opens no network ports. Analysis happens on the VM, but tool results are sent to whichever model provider Hermes is using, so anything analyzed should be treated as shared with that provider.

## Prerequisites

- A working Hermes install with MCP support
- Node.js 22.23.4+, 24.22.1+, or 26.12.1+
- A Proxmox snapshot of the VM before starting

## Procedure

### 1. Snapshot the VM

Took a Proxmox snapshot (`pre-rea`) so the whole lab could be rolled back.

### 2. Check Node.js

```bash
node -v; npm -v; which node
```

Result: Node `v26.12.1`, npm `12.0.2`, located in Hermes's bundled tools folder. This meets REA's minimum for the Node 26 line, so no separate Node install was needed.

> **Note:** Because REA installs into Hermes's bundled Node, a Hermes update that replaces that Node folder may require reinstalling REA.

### 3. Install REA

```bash
npm install --global rea-agents
rea --version                  # 6.8.0
which node
readlink -f "$(which rea)"
```

npm blocked the optional Frida install script. Frida is only used for live runtime instrumentation, which this lab intentionally excludes, so this was left as-is.

### 4. Health check

```bash
rea doctor
```

Node and the OS passed. Doctor reported "unhealthy" only because no native analysis engines were installed and Hermes wasn't registered yet. It also detected Hermes as a supported client.

### 5. Register REA with Hermes

REA's setup supports Hermes directly. I previewed the changes first, then applied them non-interactively:

```bash
rea setup --client hermes --dry-run
rea setup --client hermes --yes
```

This added an `mcp_servers.rea` entry to `~/.hermes/config.yaml`, backed up the original config to `config.yaml.rea.backup`, and installed the REA skill (agent instructions) at `~/.hermes/skills/reverse-engineer-anything`.

### 6. Fix: Node not found in Hermes's MCP environment

**Symptom:** After restarting, no REA tools were available to the agent.

**Diagnosis:** Hermes does not pass the full shell environment to MCP servers. The `rea` launcher finds Node by name (`#!/usr/bin/env node`), and Hermes's Node isn't in a minimal PATH. I reproduced this by running REA with a stripped environment:

```bash
env -i PATH=/usr/bin:/bin ~/.hermes/tools/node-26.12.1-linux-x64/bin/rea --version
# env: 'node': No such file or directory  (exit 127)
```

**Fix:** Instead of changing the command REA setup wrote, I gave the `rea` server its own PATH in the Hermes config. This keeps the entry aligned with what `rea doctor` expects.

```yaml
    env:
      PATH: "/home/clarkfaint/.hermes/tools/node-26.12.1-linux-x64/bin:/usr/local/bin:/usr/bin:/bin"
```

**Verification:**

```bash
env -i PATH="/home/clarkfaint/.hermes/tools/node-26.12.1-linux-x64/bin:/usr/local/bin:/usr/bin:/bin" \
  ~/.hermes/tools/node-26.12.1-linux-x64/bin/rea --version
# 6.8.0
```

Config backup before this change: `config.yaml.pre-path-fix`.

### 7. Verify the connection

```bash
hermes mcp list        # rea: enabled, all tools
hermes mcp test rea    # Connected (1617 ms), 229 tools discovered
```

Then I ran a live tool call from a fresh Hermes Desktop chat:

> Use your REA MCP tool for inspecting ELF notes on /usr/bin/ls. Tell me the GNU build-id and the exact tool name you called.

Result: Hermes called `mcp__rea__inspect_elf_notes` and returned GNU build-id `1fb6b102c123cbc300395919f1c5072c745dd8aa`. REA also noted that `/usr/bin/ls` is a symlink and inspected the resolved target, `/usr/lib/cargo/bin/coreutils/ls`.

> **Gotcha:** Hermes names MCP tools `mcp__<server>__<tool>`, with **double** underscores. An earlier check for `mcp_rea_` (single underscores) wrongly returned zero tools. Testing with a real tool call is more reliable than asking the agent to describe its own tool list.

### 8. Trim the tool list

REA exposes 229 tools, and every exposed tool's description is sent to the model with each message, which adds to token cost. Many of those tools also can't run on a headless server (for example, Hopper GUI tools). I used Hermes's per-server allow-list (`tools.include`) to expose only static-analysis tools that work without extra engines.

Before writing it, I had Hermes validate my proposed names against the real tool list from `hermes mcp test rea`. Five names were CLI-only commands and didn't exist as MCP tools; `evidence_export` was corrected to `export_evidence_bundle`.

Final `rea` block:

```yaml
  rea:
    command: /home/clarkfaint/.hermes/tools/node-26.12.1-linux-x64/bin/rea
    args:
      - mcp
    enabled: true
    env:
      PATH: "/home/clarkfaint/.hermes/tools/node-26.12.1-linux-x64/bin:/usr/local/bin:/usr/bin:/bin"
    tools:
      include:
        - inspect_artifact
        - extract_artifact
        - search_bytes
        - read_bytes
        - inspect_byte_entropy
        - inspect_analysis_settings
        - export_evidence_bundle
        - inspect_binary_layout
        - inspect_elf_notes
        - inspect_go_binary
        - inspect_rust_binary
        - inspect_pe_certificates
        - inspect_pe_imports
        - inspect_pe_resources
        - inspect_pe_load_config
        - inspect_signature
        - inspect_with_objdump
        - analyze_javascript_application
        - inspect_desktop_application
        - inspect_electron_fuses
        - recover_javascript_sources
        - inspect_managed_artifact
        - inspect_managed_members
        - inspect_python_pyc
        - inspect_web_network_capture
        - inspect_sqlite_database
```

| Category | Tools kept |
|---|---|
| General | artifact inspection/extraction, byte search/read, entropy, analysis settings, evidence export |
| Binaries and supply chain | binary layout, ELF notes, Go and Rust dependency metadata, PE certificates/imports/resources/load config, signatures, objdump |
| Applications | JavaScript/Electron analysis, Electron fuses, source map recovery, desktop app inspection, .NET metadata, Python bytecode |
| Data and traffic | HAR network captures, SQLite databases |

Deliberately excluded for now: Hopper/IDA/JEB integrations, Android and adb, Frida, debuggers, runtime capture (browser, Electron, process), firmware, signal decoding, and the RL environment builder.

Config backup before this change: `config.yaml.pre-allowlist`.

**Result after reboot:** 28 `mcp__rea__` tools: the 26 allow-listed tools plus `get_prompt` and `list_prompts`, Hermes's wrappers for REA's guided-investigation prompts. That's down from 229.

## First exercise: SBOM from a Rust system binary

Ubuntu 26.04 ships the Rust rewrite of coreutils (uutils), which made `ls` a convenient supply-chain test target.

> Use REA to inspect the Rust binary at /usr/lib/cargo/bin/coreutils/ls. List its embedded crate dependencies and versions, then tell me what evidence supports that and what REA couldn't determine.

**Findings:**

- REA read the 11.8 MB x86_64 ELF via `inspect_rust_binary`.
- No `.dep-v0` section was present, so there was no embedded cargo-auditable dependency list.
- No `/rustc/<hash>` path was found, so the compiler version was unknown.
- Zero crate dependencies were reported. REA explicitly stated that the absence of embedded metadata is **not** proof that the binary has no dependencies.

**Interpretation:** cargo-auditable is an opt-in build tool, and Ubuntu doesn't appear to use it for this package. The binary also appears to be stripped of compiler paths. The dependency information simply isn't in the file.

**Takeaway:** You can't always get an SBOM from the binary itself. For distro-packaged software, package metadata is often the better source. Debian/Ubuntu Rust packages typically record the crates they were statically built against in a `Static-Built-Using` (older: `X-Cargo-Built-Using`) field.

**Follow-up (to complete):**

```bash
dpkg -S /usr/lib/cargo/bin/coreutils/ls     # which package owns the binary
dpkg -s <package-name>                      # look for a Built-Using field
```

| Method | Result |
|---|---|
| Binary inspection (REA `inspect_rust_binary`) | No embedded dependency metadata |
| Package metadata (`dpkg -s`) | _TBD_ |

## Security considerations

- **Prompt injection:** Analysis results feed text from the target file (strings, code, payloads) directly to the model. A hostile file could contain text written to steer the agent. This VM holds API credentials and a Slack connection, and Hermes has shell access.
- **Rule for this VM:** Static analysis only, on trusted files, practice targets, or software I'm authorized to examine. The allow-list keeps runtime tools (Frida, process/browser capture) out of REA's exposed tools, but it doesn't restrict Hermes's shell, so this is a usage rule rather than a technical control.
- **Untrusted samples belong elsewhere:** Anything untrusted or that needs to be executed should go to a separate, credential-free analysis VM on an isolated network segment.
- **Data handling:** Tool output goes to the configured model provider. Don't analyze anything I'm not comfortable sharing with that provider.
- **Legal:** REA's own disclaimer limits it to lawful research, interoperability, education, preservation, and software you own or are authorized to examine. Many commercial EULAs prohibit reverse engineering, so client work requires written authorization.

## Rollback

| Backup | Restores to |
|---|---|
| Proxmox snapshot `pre-rea` | Entire VM before this lab |
| `~/.hermes/config.yaml.rea.backup` | Config before REA registration |
| `~/.hermes/config.yaml.pre-path-fix` | REA registered, before the PATH fix |
| `~/.hermes/config.yaml.pre-allowlist` | PATH fixed, before tool trimming |

> Re-running `rea setup` may rewrite the `rea` block and drop the `env` and `tools` sections. Re-check the config after any REA update.

## Lessons learned

1. **Preview before applying.** `rea setup --dry-run` showed exactly what would change before anything was written.
2. **MCP servers don't inherit your shell.** Agent frameworks often launch MCP servers with a minimal environment. Reproducing that with `env -i` isolated the problem in one command.
3. **Test with real calls.** Asking an agent to describe its own tools gave misleading answers twice. Making it call a specific tool was definitive.
4. **Validate names against the source of truth.** Five of my proposed tool names were wrong; checking against `hermes mcp test rea` caught them before they went into the config.
5. **Less is more for agent tools.** Trimming 229 tools to 26 reduces token cost and keeps the agent focused on tools that actually work in this environment.
6. **Absence of evidence is not evidence of absence.** The empty Rust dependency result was still a useful finding, and it showed when binary inspection works for SBOMs and when it doesn't.

## Next steps

- [ ] Complete the `dpkg` comparison for the Rust coreutils SBOM exercise
- [ ] Install headless Ghidra (and a JDK) for native decompilation
- [ ] Build a separate, credential-free analysis VM on an isolated network for untrusted samples
- [ ] Vendor-risk exercise: capture a desktop app's traffic as a HAR file and analyze it with `inspect_web_network_capture`
- [ ] Signature exercise: verify Authenticode signers on a set of Windows installers with `inspect_pe_certificates`

## References

- REA: https://github.com/morluto/rea · https://rea.tools
- Hermes Agent MCP docs: https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp
- Hermes MCP config reference: https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference
