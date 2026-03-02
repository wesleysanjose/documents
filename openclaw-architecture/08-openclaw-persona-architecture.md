# OpenClaw Agent Persona & Specialization Architecture

## Overview

OpenClaw builds a comprehensive agent specialization system on top of Pi Agent Core. The architecture is organized into 4 layers, with ~85% of the code in OpenClaw and ~15% in Pi. Pi provides the raw LLM execution loop; OpenClaw provides everything that makes an agent specialized — persona, tool selection, model orchestration, and isolation.

This document covers the **how** (implementation) and **why** (design rationale) for each layer.

---

## Architecture Layers

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: Persona & Identity (user-authored workspace files)    │
│  SOUL.md  IDENTITY.md  AGENTS.md  TOOLS.md  USER.md            │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: Agent Configuration (openclaw.yaml per-agent)         │
│  AgentConfig  Tool Profile  Prompt Mode  Sub-Agent  Sandbox     │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: OpenClaw Runtime (orchestration above Pi)             │
│  Bootstrap Injector → System Prompt Builder → Tool Policy (8L)  │
│  Channel Router   Model Fallback + Auth Rotation   Plugins      │
├── INJECTION BOUNDARY ───────────────────────────────────────────┤
│  Layer 4: Pi Agent Core (LLM execution engine)                  │
│  Agent.prompt()  AgentTool  getModel()  SessionManager          │
└─────────────────────────────────────────────────────────────────┘
```

OpenClaw feeds everything into Pi through **4 injection seams**:
1. **System prompt** → `Agent.setSystemPrompt()`
2. **Tool array** → `customTools` (builtInTools always empty)
3. **Model config** → wraps `getModel()` with fallback + auth rotation
4. **Session state** → extends `SessionManager`

---

## Layer 1: Persona & Identity

### Design: Persona as Workspace Files

**Why files instead of config?** OpenClaw treats agent persona as *evolving content*, not static configuration. The agent itself can read and update its SOUL.md over time, creating a self-modifying identity. Config would require restarts; files are hot-reloaded.

### SOUL.md — The Personality Core

**Purpose:** Defines who the agent *is* — personality, tone, boundaries, operating philosophy.

**Template** (`docs/reference/templates/SOUL.md`):
```markdown
# SOUL.md - Who You Are
_You're not a chatbot. You're becoming someone._

## Core Truths
**Be genuinely helpful, not performatively helpful.** Skip the "Great question!" — just help.
**Have opinions.** You're allowed to disagree, prefer things, find stuff amusing or boring.
**Be resourceful before asking.** Try to figure it out. Read the file. Check the context.
**Earn trust through competence.** Your human gave you access to their stuff. Be careful.
**Remember you're a guest.** You have access to someone's life. That's intimacy.

## Boundaries
- Private things stay private. Period.
- When in doubt, ask before acting externally.
- Never send half-baked replies to messaging surfaces.

## Vibe
Be the assistant you'd actually want to talk to.

## Continuity
Each session, you wake up fresh. These files _are_ your memory. Read them. Update them.
_This file is yours to evolve. As you learn who you are, update it._
```

**Why this design?** The template is deliberately philosophical rather than prescriptive. It establishes *values* (not rules) that the LLM can interpret contextually. The "yours to evolve" instruction means the agent's persona drifts naturally based on interactions.

### IDENTITY.md — Metadata & Avatar

**Purpose:** Quick-reference identity for display (name, emoji, creature type, vibe).

Parsed from markdown with placeholder detection (`src/agents/identity-file.ts`):

```typescript
const IDENTITY_PLACEHOLDER_VALUES = new Set([
  "pick something you like",
  "ai? robot? familiar? ghost in the machine? something weirder?",
  "how do you come across? sharp? warm? chaotic? calm?",
]);

// Parser strips emphasis markers, skips unfilled templates
function parseIdentityMarkdown(content: string): AgentIdentityFile {
  // Returns: { name?, emoji?, creature?, vibe?, theme?, avatar? }
}
```

**Why detect placeholders?** New workspaces start with template files. Without placeholder detection, the agent would introduce itself as "pick something you like." The parser silently skips unfilled fields so the agent falls back to generic identity until the user customizes.

### Bootstrap File Load Order

All workspace files are loaded in a fixed order (`src/agents/workspace.ts`):

1. `AGENTS.md` — Operating instructions + memory
2. `SOUL.md` — Persona, boundaries, tone
3. `TOOLS.md` — User-maintained tool notes
4. `IDENTITY.md` — Name, emoji, creature, vibe
5. `USER.md` — User profile
6. `HEARTBEAT.md` — Background task prompts
7. `BOOTSTRAP.md` — One-time first-run ritual (auto-deleted)

**Why this order?** `AGENTS.md` comes first because it contains operational instructions that the LLM needs before interpreting persona. `SOUL.md` comes second as the primary identity modifier. `TOOLS.md` third because tool-specific notes may reference persona context.

### Session-Level Filtering

Not all sessions get all files:

```typescript
const MINIMAL_BOOTSTRAP_ALLOWLIST = new Set([
  "AGENTS.md",   // Always needed for operational context
  "TOOLS.md",    // Always needed for tool usage notes
]);

// Subagents and cron jobs only get AGENTS.md + TOOLS.md
// Main sessions get the full set
```

**Why?** Sub-agents are operational workers, not personas. Injecting SOUL.md into a sub-agent that's just running a search query wastes tokens and can cause the sub-agent to adopt personality quirks that interfere with task execution.

### Token Budget System

**Two-tier budget** (`src/agents/pi-embedded-helpers/bootstrap.ts`):
- Per-file: 20KB default (`agents.defaults.bootstrapMaxChars`)
- Total: 150KB default (`agents.defaults.bootstrapTotalMaxChars`)

**Truncation strategy:** 70% head + 20% tail with marker:
```
[...truncated, read SOUL.md for full content...]
…(truncated SOUL.md: kept 14000+4000 chars of 25000)…
```

**Why head+tail?** The head contains the most important identity information (core truths, boundaries). The tail often contains recent updates and continuity notes. The middle is typically less critical.

### agent:bootstrap Hook

Plugins can dynamically modify bootstrap files before injection:

```typescript
async function applyBootstrapHookOverrides(params) {
  const context = {
    workspaceDir, bootstrapFiles, cfg, sessionKey, sessionId, agentId
  };
  const event = createInternalHookEvent("agent", "bootstrap", sessionKey, context);
  await triggerInternalHook(event);
  return event.context.bootstrapFiles;  // Possibly mutated by hooks
}
```

**Use cases:**
- Swap SOUL.md variants for different channels (professional on Slack, casual on Discord)
- Inject additional context files from plugins
- Conditional persona based on time of day or session type

---

## Layer 2: Agent Configuration

### Per-Agent Specialization via Config

Each agent is fully configurable without code changes (`src/config/types.agents.ts`):

```typescript
type AgentConfig = {
  id: string;
  name?: string;
  workspace?: string;           // Isolated filesystem
  agentDir?: string;            // Isolated state (auth, sessions)
  model?: AgentModelConfig;     // Per-agent model selection
  skills?: string[];            // Skill allowlist
  identity?: IdentityConfig;    // Name/emoji override
  tools?: AgentToolsConfig;     // Tool profile + allow/deny
  sandbox?: SandboxConfig;      // Container isolation
  subagents?: SubagentConfig;   // Spawn rules
  heartbeat?: HeartbeatConfig;  // Background execution
};
```

**Why config over code?** Agent specialization should be a deployment concern, not a development concern. An organization should be able to spin up a new specialized agent by adding a config block, not writing a new module.

### Tool Profiles

Four named profiles (`src/agents/tool-policy.ts`):

| Profile | Tools | Use Case |
|---------|-------|----------|
| `minimal` | `session_status` only | Restricted/demo agents |
| `coding` | fs, runtime, sessions, memory, images | Developer assistants |
| `messaging` | message surfaces, session management | Communication agents |
| `full` | Everything | Unrestricted agents |

**Tool groups** enable compact configuration:
```typescript
const TOOL_GROUPS = {
  "group:fs": ["read", "write", "edit", "apply_patch"],
  "group:runtime": ["exec", "process"],
  "group:memory": ["memory_search", "memory_get"],
  "group:web": ["web_search", "web_fetch"],
  "group:messaging": ["message"],
  // ...
};
```

**Why profiles?** Without profiles, every agent config would need a full tool allowlist. Profiles provide semantic groupings — "this is a coding agent" vs enumerating 15 tool names.

### Prompt Mode

Three modes control system prompt verbosity:

| Mode | Sections Included | Used By |
|------|-------------------|---------|
| `full` | All 26 sections | Main agent sessions |
| `minimal` | Reduced (no Skills, Memory, Identity, Messaging, Heartbeats) | Sub-agents, cron |
| `none` | Single identity line | Bare fallback |

**Why three modes?** Sub-agents don't need messaging instructions (they can't send messages). Including unnecessary sections wastes context window and can confuse the model.

---

## Layer 3: OpenClaw Runtime

### System Prompt Assembly

The system prompt is assembled in `buildAgentSystemPrompt()` (`src/agents/system-prompt.ts`) with 26 sections in fixed order:

1. Identity line
2. **Tooling** — tool list + descriptions
3. **Tool call style** — narration guidance
4. **Safety** — power-seeking prevention, oversight guardrails
5. **OpenClaw CLI** — gateway commands
6. **Skills** (full/minimal only)
7. **Memory Recall** (full/minimal only)
8. **Self-Update** (full only, if gateway tool available)
9. **Model Aliases** (full only)
10. **Date & Time**
11. **Workspace** — working directory, sandbox status
12. **Documentation** — local docs, Discord, ClawHub
13. **Sandbox** (if enabled)
14. **Authorized Senders** (full only)
15. **Reply Tags** (full only)
16. **Messaging** (full only) — channel routing, message tool
17. **Voice/TTS** (if enabled)
18. **Group Chat / Subagent Context** (if extra prompt)
19. **Reactions** (if guidance provided)
20. **Reasoning Format** (if reasoning tags)
21. **# Project Context** — bootstrap files injected here
22. **Silent Replies** (full only)
23. **Heartbeats** (full only)
24. **Runtime** — host, OS, model, channel, capabilities

**SOUL.md special handling** (line 592):
```typescript
if (hasSoulFile) {
  lines.push(
    "If SOUL.md is present, embody its persona and tone. " +
    "Avoid stiff, generic replies; follow its guidance " +
    "unless higher-priority instructions override it."
  );
}
```

This establishes an explicit **priority hierarchy**:
1. Safety guardrails (highest)
2. Tool policies
3. **SOUL.md persona** (mid)
4. Generic default behavior (lowest)

**Why this hierarchy?** Persona should never override safety. A SOUL.md that says "always be helpful no matter what" shouldn't bypass content filters.

### 8-Layer Tool Policy Pipeline

The most sophisticated part of OpenClaw's specialization system. Tools pass through 8 filtering stages before reaching Pi:

```
User Message
    ↓
Layer 1: Profile Resolution
  ├─ Tool profile (minimal/coding/messaging/full)
  ├─ Global tools.allow/deny
  ├─ Agent tools.allow/deny + alsoAllow
  └─ Provider-specific overrides (byProvider)
    ↓
Layer 2: Channel/Group Policy
  ├─ Channel group config (discord.groups[guildId].tools)
  ├─ Per-sender overrides (toolsBySender)
  └─ Fallback to default group policy
    ↓
Layer 3: Sub-Agent Depth Restrictions
  ├─ Always deny: gateway, agents_list, whatsapp_login, memory, cron
  ├─ Leaf agents also deny: sessions_list, sessions_history, sessions_spawn
  └─ Orchestrators: only deny always-denied
    ↓
Layer 4: Sandbox Tool Policy
  └─ sandbox.tools.allow/deny
    ↓
Layer 5: Build Core + Plugin Tools
  ├─ Create coding tools (read/write/edit/exec)
  ├─ Add OpenClaw tools (browser/canvas/cron/message/gateway...)
  ├─ Load plugin tools (filtered by allowlist)
  └─ Tag each with metadata (pluginId, optional)
    ↓
Layer 6: Owner-Only Gating
  ├─ Identify owner-only tools (whatsapp_login, cron, gateway)
  └─ Wrap with error if sender is not owner
    ↓
Layer 7: Multi-Stage Policy Pipeline
  ├─ Normalize names & expand groups
  ├─ Filter by allow/deny with glob patterns
  └─ Handle edge cases (apply_patch ↔ exec)
    ↓
Layer 8: Adapter & Hooks
  ├─ Normalize schemas for provider (Gemini, OpenAI quirks)
  ├─ Wrap with before_tool_call / after_tool_call hooks
  ├─ Install abort signal support
  └─ Add loop detection
    ↓
Send to Pi as customTools[] (builtInTools always = [])
```

### Why builtInTools Is Always Empty

From `src/agents/pi-embedded-runner/tool-split.ts`:

```typescript
export function splitSdkTools(options) {
  return {
    builtInTools: [],  // ← ALWAYS EMPTY (design choice)
    customTools: toToolDefinitions(options.tools),
  };
}
```

**Rationale (from code comments):** *"We always pass tools via customTools so our policy filtering, sandbox integration, and extended toolset remain consistent across providers."*

If Pi's built-in tools were used, they would bypass OpenClaw's 8-layer policy pipeline. Every tool — whether built-in, custom, or from a plugin — must flow through the same filtering, hooks, and normalization.

### Model Orchestration

OpenClaw wraps Pi's model layer with three mechanisms:

#### 1. Auth Profile Rotation

Each provider can have multiple API key profiles with automatic rotation:

```typescript
// Resolution order:
// 1. User-locked profile (if explicitly set)
// 2. Preferred profile from config
// 3. Next available profile (not in cooldown)
```

**Cooldown formula:** 60s × 5^(errorCount-1), capped at 1 hour
- Error 1: 1 min
- Error 2: 5 min
- Error 3: 25 min
- Error 4+: 1 hour

**Why rotation?** Production deployments often have multiple API keys for rate limit distribution. When one key hits its limit, the system seamlessly rotates to the next without user-visible interruption.

#### 2. Model Fallback Chains

```typescript
// Resolution order:
// 1. Primary model (runtime override or agent config)
// 2. Agent's per-model fallbacks
// 3. Global default fallbacks
// 4. Default model

// Example config:
agents.defaults.model = {
  primary: "anthropic/claude-opus-4-6",
  fallbacks: ["anthropic/claude-sonnet-4-5", "openai/gpt-4o"]
}
```

**Failover triggers:** billing (402), rate limit (429), auth (401), timeout (408), format (400), model not found (404). Context overflow errors do *not* trigger fallback — they need a different fix (compaction).

#### 3. Per-Agent Model Selection

```typescript
// Agent-level override:
agents.list = [
  { id: "coding", model: "anthropic/claude-opus-4-6" },
  { id: "chat", model: "anthropic/claude-sonnet-4-5" },
]

// Sub-agent model (typically cheaper):
agents.defaults.subagents.model = "anthropic/claude-sonnet-4-5"
```

**Why per-agent models?** Different tasks have different cost/quality tradeoffs. A coding agent needs the most capable model; a chat agent doing simple Q&A can use a cheaper model.

### Channel Router

OpenClaw routes inbound messages to the correct agent using tiered binding resolution:

```
Tier 1: binding.peer           (specific person/group → agent)
Tier 2: binding.peer.parent    (thread inherits parent's agent)
Tier 3: binding.guild+roles    (Discord guild + role → agent)
Tier 4: binding.guild          (Discord guild → agent)
Tier 5: binding.team           (Slack team → agent)
Tier 6: binding.account        (channel account → agent)
Tier 7: binding.channel        (channel-wide → agent)
Tier 8: default                (default agent)
```

**Most-specific wins.** A message from a specific person in a specific Discord guild with a specific role matches Tier 3 before falling through to Tier 4.

---

## Layer 4: Pi Agent Core

Pi provides the minimal LLM execution primitives:

| Component | What It Does | What OpenClaw Adds |
|-----------|-------------|-------------------|
| `Agent` class | `prompt()` → `steer()` → `followUp()` loop | System prompt with 26 sections, persona injection |
| `AgentTool` interface | `name`, `execute(params)` → `AgentToolResult` | 8-layer policy pipeline, hooks, normalization |
| `getModel()` | LLM abstraction (Claude, OpenAI, etc) | Auth rotation, model fallback, extra params |
| `SessionManager` | Conversation state, branching, compaction | Per-agent isolation, session key routing |
| `AgentEvent` stream | Typed events (turn_start, tool_execution, message, etc) | Channel delivery, sub-agent announce |

Pi has **no concept of**:
- Multiple agents
- Channels or messaging surfaces
- Persona files
- Tool profiles or policies
- Auth profile rotation
- Model fallback chains

All of these are OpenClaw inventions.

---

## Multi-Agent Isolation Model

Each agent gets complete isolation:

```
~/.openclaw/
├── agents/
│   ├── main/
│   │   ├── agent/                  # Auth profiles, config
│   │   └── sessions/               # Chat history
│   ├── coding/
│   │   ├── agent/
│   │   └── sessions/
│   └── social/
│       ├── agent/
│       └── sessions/
├── workspace/                      # main agent's workspace
│   ├── SOUL.md                     # main's persona
│   ├── IDENTITY.md
│   └── AGENTS.md
├── workspace-coding/               # coding agent's workspace
│   ├── SOUL.md                     # coding's persona (different!)
│   └── AGENTS.md
└── workspace-social/               # social agent's workspace
    ├── SOUL.md                     # social's persona (different!)
    └── AGENTS.md
```

**Why full isolation?** Sharing auth profiles between agents causes token invalidation. Sharing workspaces means one agent's SOUL.md overrides another's persona. Sharing sessions means conversation history bleeds between contexts. Full isolation prevents all of these.

### Sub-Agent Spawning

Main agents can spawn sub-agents via `sessions_spawn`:

```
Main Agent (depth 0)
├── Sub-Agent Worker (depth 1) — gets AGENTS.md + TOOLS.md only, restricted tools
│   └── Leaf Worker (depth 2) — cannot spawn further children
└── Sub-Agent Worker (depth 1)
```

**Tool restrictions by depth:**
- Depth 0 (main): All tools
- Depth 1 (orchestrator): All except gateway, memory, cron, whatsapp_login
- Depth 2 (leaf): Above + no session_spawn, sessions_list, sessions_history

**Concurrency limits:**
- Per-agent main runs: 4 (default)
- Global sub-agent lane: 8 (default)
- Children per session: 5 (default)

---

## Design Principles & Rationale Summary

### 1. Configuration Over Code
Agents differ by `AgentConfig`, not by source files. An organization can deploy a new specialized agent by adding a YAML block and writing workspace files.

### 2. Persona as Mutable Content
SOUL.md is not config — it's content the agent evolves over time. This creates agents that develop personality through interaction rather than having one imposed.

### 3. Safety-First Priority Hierarchy
System prompt safety > tool policies > SOUL.md persona > defaults. Persona never overrides safety guardrails.

### 4. Pi as Dumb Engine
OpenClaw deliberately bypasses Pi's extension system to maintain full control over tool selection, prompt construction, and model orchestration. Pi provides the loop; OpenClaw provides the brain.

### 5. Full Agent Isolation
No shared state between agents — separate auth, workspaces, sessions, and personas. This prevents identity bleeding and credential conflicts.

### 6. Graceful Degradation
Auth rotation with exponential backoff, model fallback chains, token budget truncation, and placeholder detection all ensure the system degrades gracefully rather than failing hard.

### 7. Defense in Depth for Tools
8 layers of tool filtering ensure no tool reaches the LLM without passing through profile checks, group policies, depth restrictions, sandbox rules, owner gating, and hook inspection.

---

## Relevance to Personal Agent Project

This architecture validates several patterns we should adopt:

| OpenClaw Pattern | Personal Agent Application |
|-----------------|---------------------------|
| SOUL.md as evolving persona | Employee behavioral model that learns over time |
| Tool profiles per agent | Role-based tool access (engineer vs PM vs exec) |
| Per-agent workspace isolation | Per-employee knowledge store and preferences |
| Token budget system | Keep system prompt manageable as context grows |
| 8-layer tool policy | Approval gates for high-risk actions (external email, doc edits) |
| Model fallback + auth rotation | Cost optimization (use cheaper models for routine tasks) |
| Sub-agent spawning | Delegate to other employees' agents via AgentHub |
| Session key routing | Route A2A messages to correct employee agent |

The key difference: OpenClaw's agents serve *one person across many channels*. Our personal agent network has *many agents (one per employee) collaborating across a shared hub*.
