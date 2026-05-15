# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is a **Microsoft Copilot Studio (MCS)** agent project. All configuration is stored as `.mcs.yml` files — there is no Python, JavaScript, or traditional build system. The agent is deployed and managed entirely through Microsoft Copilot Studio, not a local runtime.

## Agent Overview

**IBM Top Products Finder** (`cr697_ibmTopProductsFinder`) — An agent that:
1. Browses ibm.com and xynotech.com to retrieve top 10 products from each
2. Uses a sub-agent (`Agent/agent.mcs.yml`) to independently re-evaluate and select the top 3 from those results
3. Emails the final output to `ansar.muhammad@10pearls.com` via Office 365 Outlook

## File Format

All components use `.mcs.yml` — a Microsoft Copilot Studio YAML schema. Key fields:
- `mcs.metadata.componentName` — human-readable display name
- `kind` — component type (e.g., `AdaptiveDialog`, `TaskDialog`, `GptComponentMetadata`)
- `beginDialog` — entry point and logic flow for topics/actions

## Architecture

```
IBM Top Products Finder/
├── agent.mcs.yml              # Root agent: instructions, model (GPT5Chat), web browsing enabled
├── settings.mcs.yml           # Deployment: channels (Teams, M365 Copilot), auth, publish settings
├── connectionreferences.mcs.yml  # External connectors (Office365, Teams, OneDrive, M365 MCPs)
├── trigger/                   # Email arrival trigger (Office 365 Outlook, Power Automate flow)
├── agents/
│   ├── topic.WorkIQUserPreview.mcs.yml    # MCP agent for user memory/context
│   ├── topic.WorkIQCopilotPreview.mcs.yml # M365 Copilot MCP integration
│   └── Agent/agent.mcs.yml               # Sub-agent: re-selects top 3 independently
├── actions/
│   ├── Office365Outlook-SendanemailV2.mcs.yml  # Send email action
│   └── WorkIQ*MCP-*.mcs.yml                   # OneDrive, Mail, Word, Teams MCP connectors
├── topics/                    # System topics (Greeting, Fallback, Escalate, etc.)
├── workflows/                 # Power Automate flow definition (email trigger)
└── .mcs/                      # Internal MCS state (botdefinition.json, gitignore, tokens)
```

### Key Design Decisions

- **Dual-agent pattern**: The root agent fetches top 10; the `Agent` sub-agent independently selects top 3 without being biased by the root's ranking (`agents/Agent/agent.mcs.yml`).
- **Trigger**: Conversations can be initiated by an incoming email (Power Automate flow in `workflows/`) or directly via Teams/M365 Copilot channels.
- **MCP connectors**: Work IQ integrations (Mail, Teams, OneDrive, Word, User memory) are all MCP-based (`shared_a365*mcp` connectors).
- **`.mcs/.gitignore`** ignores everything — meaning `.mcs/` internal state should not be committed. Only `.mcs.yml` files are source-controlled.

## Deployment

This agent is deployed via Microsoft Copilot Studio UI — there is no CLI deploy command. To publish changes:
1. Edit `.mcs.yml` files in the repository
2. Import the solution into Copilot Studio (or use the MCS VS Code extension to sync)
3. Publish from the Copilot Studio portal

Published on: `2026-04-10`. Schema name: `cr697_ibmTopProductsFinder`. Language: English (1033).

## Channels

- Microsoft Teams (`MsTeams`)
- Microsoft 365 Copilot (`Microsoft365Copilot`)
