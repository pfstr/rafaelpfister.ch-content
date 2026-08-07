---
title: "Claude and Claude Code: AI assistance for technical work"
blatt: "claude"
description: "An overview of Claude: Anthropic's AI model, access paths from chat to API, Claude Code as an agent for the terminal and the codebase, and the security questions that arise when running it on self-managed infrastructure."
fakten:
  - { label: "Vendor", wert: "Anthropic", href: "https://www.anthropic.com/" }
  - { label: "Product family", wert: "Claude (chat, API) and Claude Code (CLI agent)" }
  - { label: "Documentation", wert: "docs.claude.com", href: "https://docs.claude.com/" }
  - { label: "Access paths", wert: "Web/app, API, IDE integrations, terminal" }
  - { label: "Typical uses", wert: "Code, scripts, analysis, documentation, automation" }
  - { label: "Operational questions", wert: "Data egress, secrets, agent permissions" }
werbung: ["newsletter"]
ctaThemen: ["claude"]
---

# Claude and Claude Code: AI assistance for technical work

Claude is Anthropic's family of large language models. Two forms are primarily of interest to technical users: the assistant for analysis and writing work via chat and API, and **Claude Code**, an agent that works in the terminal, reads and writes files, runs commands, and thereby takes on entire work steps instead of individual answers.

## From chat to agent

The difference between a chatbot and an agent lies in its scope for action. A chat answers questions; an agent such as Claude Code is given tools: it navigates a codebase, modifies files, runs builds and tests, calls APIs, and works iteratively until a goal is reached. It is directed through natural language plus project context, such as convention files in the repository; control remains with the user through permission levels and confirmation steps.

## Fields of application in administration and operations

Recurring patterns have become established in system administration: **scripts and automation** (PowerShell, Bash, Python to specification), **analysis** of logs, headers, and configurations, **documentation** derived from existing systems, and **guided procedures** in which the agent executes a process step by step and checks intermediate results along the way. For reproducible tasks, instructions can be stored as reusable building blocks, which turns one-off prompts into reviewed procedures.

## Security and operational questions

Running an AI agent on self-managed infrastructure raises the same questions as any powerful tool, plus a few of its own. **Data flow**: input leaves the local system in the direction of the API; whatever the agent reads, it should also be permitted to read. **Secrets**: credentials do not belong in prompts or in files that are read in, but in the mechanisms intended for them. **Permissions**: an agent with shell access operates with the rights of its user; separate accounts, restricted environments, and the built-in confirmation levels limit the damage caused by mistakes. **Auditability**: session logs make it verifiable what the agent has done. Running Claude Code on a server is sensibly combined with the usual system hardening, from SSH keys to the firewall.
