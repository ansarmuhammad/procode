# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is a **Microsoft Copilot Studio (MCS)** agent project. All configuration is stored as `.mcs.yml` files — there is no Python, JavaScript, or traditional build system. The agent is deployed and managed entirely through Microsoft Copilot Studio, not a local runtime.

## Agent Overview

**IBM Top Products Finder** (`cr697_ibmTopProductsFinder`) — An agent with two distinct behaviours:

1. **Product search** (user-initiated): Browses ibm.com and xynotech.com to retrieve top 10 products from each, uses a sub-agent (`agents/Agent/agent.mcs.yml`) to independently re-select the top 3, then emails the result to `ansar.muhammad@10pearls.com`.
2. **Rafay email archiver** (automated trigger): When `rafay.moin@10pearls.com` sends an email, the Power Automate flow fires, passes the email to the agent, which (a) creates a Word document named `Copilot <Subject>.docx` (heading: `Copilot <Subject>`, then From / Date / Body) and saves it to OneDrive root, then (b) appends a `<UTC datetime> | <Subject>` line to `Agent Activity Log.docx` in OneDrive root (creating the file with a header row if it does not yet exist).

### Mandatory start-of-conversation behavior

Before responding to **any** user request, the root agent is instructed to list files in two OneDrive folders via the Work IQ OneDrive tool and print them as numbered lists:
- `PSTD - doc files`
- `PSTD - pdf deck`

Only after both lists are printed does the agent handle the actual request. This is encoded in the `instructions:` block of `agent.mcs.yml` (the "PSTD OneDrive File Listing (Always Run First)" section) and will fire on every turn during testing.

## File Format

All components use `.mcs.yml` — a Microsoft Copilot Studio YAML schema. Key fields:
- `mcs.metadata.componentName` — human-readable display name
- `kind` — component type discriminator. Common values in this repo:
  - `GptComponentMetadata` — the root agent (`agent.mcs.yml`)
  - `AgentDialog` — sub-agents exposed as tools (e.g. `agents/Agent/agent.mcs.yml`, with `beginDialog.kind: OnToolSelected` meaning the root agent invokes it as a tool)
  - `AdaptiveDialog` / `TaskDialog` — topics and actions
  - `ExternalTriggerConfiguration` — the email trigger wired to a Power Automate flow
- `beginDialog` — entry point and logic flow for topics/actions

### Where the behavior actually lives

Most of the agent's behavior is **not** in topics or actions — it is a single multi-section markdown prompt inside the `instructions: |-` YAML block of `agent.mcs.yml`. Editing agent behavior almost always means editing that string. Preserve the `|-` block indentation; YAML is whitespace-sensitive.

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
- **Trigger**: The Power Automate flow (`workflows/`) already filters `from: rafay.moin@10pearls.com` at the connector level — no topic-level filtering needed. The workflow message instructs the agent what action to take before passing `@{triggerBody()}`. Direct conversations arrive via Teams/M365 Copilot channels.
- **MCP connectors**: Work IQ integrations (Mail, Teams, OneDrive, Word, User memory) are all MCP-based (`shared_a365*mcp` connectors). These are wired in `connectionreferences.mcs.yml` and referenced by logical name in each action file.
- **`.mcs/` is not source-controlled**: `.mcs/.gitignore` ignores everything in that directory. Only `.mcs.yml` files belong in git.

## Deployment

Changes to `.mcs.yml` files must be pushed into Copilot Studio manually — there is no local run or test command.

**From VS Code (recommended):** Install the **Power Platform Tools** extension (Microsoft). It bundles `pac` for macOS, adds a sidebar, and provides a Push to Copilot Studio button after signing in.

**From the web UI:** Open [copilotstudio.microsoft.com](https://copilotstudio.microsoft.com) → IBM Top Products Finder → Settings → Instructions, paste updated content, Save, then Publish.

> **Note:** `pac` CLI via `dotnet tool install` is Windows-only — the NuGet package ships no macOS binary. Use the VS Code extension on macOS.

Published on: `2026-04-10`. Schema name: `cr697_ibmTopProductsFinder`. Environment: `org3e6b68e9`. Language: English (1033).

## Channels & Auth

- Channels: Microsoft Teams (`MsTeams`), Microsoft 365 Copilot (`Microsoft365Copilot`)
- `accessControlPolicy: Any`, `authenticationMode: Integrated`, `authenticationTrigger: Always` — every conversation re-authenticates against the integrated tenant identity.
- `GenerativeActionsEnabled: true`, `useModelKnowledge: true`, `isSemanticSearchEnabled: true`, `isFileAnalysisEnabled: true` — generative-AI features all on; recognizer is `GenerativeAIRecognizer` (no rule-based intent matching).
