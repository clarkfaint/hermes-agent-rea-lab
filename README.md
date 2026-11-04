# Hermes Agent + REA Lab

A self-hosted AI agent on Proxmox, extended with reverse-engineering tools for software supply-chain and vendor-risk analysis.

This repo documents how I built and extended the lab, including the problems I hit and how I diagnosed them.

## Labs

| # | Lab | What it covers | Status |
|---|---|---|---|
| 01 | [Hermes Agent Deployment](labs/01-hermes-agent-deployment/README.md) | Deploying Nous Research's Hermes Agent in an Ubuntu Server VM on Proxmox, with access through Hermes Desktop and Slack | Complete |
| 02 | [REA Integration](labs/02-rea-integration/README.md) | Connecting REA as an MCP server, fixing its runtime environment, trimming 229 tools to 26 static-analysis tools, and a first SBOM exercise | In progress |

## Lab environment

| Component | Details |
|---|---|
| Hypervisor | Proxmox VE 9.2.2 |
| Agent VM | Ubuntu Server 26.04 (headless) |
| Agent | [Hermes Agent](https://hermes-agent.nousresearch.com/) (Nous Research) |
| Agent tools | [REA (Reverse Engineer Anything)](https://github.com/morluto/rea) 6.8.0 via MCP |

## Why this lab

Many security and compliance questions about software come down to three things:

- What does this application contain, and what does it depend on?
- Is this executable properly signed, and by whom?
- What does this app send over the network?

This lab gives an AI agent the tools to answer those questions from evidence in the files, and documents where that approach works and where it falls short.

## Ground rules

- Analysis is limited to software I own, practice targets, and software I'm authorized to examine.
- My rule for the agent VM is static analysis only. This is a usage rule, not a technical control: the allow-list limits which REA tools are exposed, but Hermes still has shell access. Untrusted or executed samples belong on a separate, isolated, credential-free VM.
- Tool output is sent to the configured model provider, so nothing sensitive is analyzed here.
