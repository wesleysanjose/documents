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

### Bootstrap Context Mode (NEW)

A new `BootstrapContextMode` controls how much bootstrap context different run types receive:

```typescript
type BootstrapContextMode = "full" | "lightweight";
type BootstrapContextRunKind = "default" | "heartbeat" | "cron";
```

| Run Kind | Mode | Bootstrap Files Loaded |
|----------|------|----------------------|
| Default (main session) | full | All workspace files |
| Heartbeat | lightweight | Only `HEARTBEAT.md` |
| Cron | lightweight | Empty (no bootstrap files) |

**Why?** Heartbeat and cron runs are automated background tasks that don't need persona context. Loading full bootstrap for a cron job wastes tokens and adds latency. The lightweight mode cuts bootstrap overhead to near-zero for automated runs.

### Bootstrap Cache Per Session Key (NEW)

Bootstrap files are now cached per `sessionKey` via `getOrLoadBootstrapFiles()`:

```typescript
const cache = new Map<string, WorkspaceBootstrapFile[]>();
// First call loads from disk, subsequent calls for same session return cached
```

**Why?** Prevents repeated filesystem I/O for the same session's bootstrap files, especially important during multi-turn conversations where bootstrap is re-evaluated on each turn.

### Bootstrap Boundary Hardening (NEW)

Workspace bootstrap reads now use guarded boundary-file operations:

```typescript
const opened = await openBoundaryFile({
  absolutePath: src,
  rootPath: seed,
  boundaryLabel: "sandbox seed workspace",
});
if (!opened.ok) continue; // Fail gracefully on boundary violation
```

**Why?** Prevents symlink/directory traversal attacks when seeding sandbox workspaces. A malicious SOUL.md path like `../../../etc/passwd` is caught and rejected at the boundary check.

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

### Data-Driven Tool Catalog with Provenance (NEW)

A major architectural shift replaced ad-hoc tool-policy helpers with a centralized `tool-catalog.ts` (322 LOC):

```typescript
type CoreToolDefinition = {
  id: string;
  label: string;
  description: string;
  sectionId: string;
  profiles: ToolProfileId[];           // Which profiles include this tool
  includeInOpenClawGroup?: boolean;
};

// 11 organized sections:
const CORE_TOOL_SECTION_ORDER = [
  "fs", "runtime", "web", "memory", "sessions",
  "ui", "messaging", "automation", "nodes", "agents", "media"
];
```

**Gateway protocol exposure:** New `gateway.tools-catalog` method lets mobile/web clients query the authoritative tool catalog dynamically, eliminating hardcoded tool lists in UIs.

**Plugin-only allowlist stripping:** A safety mechanism prevents accidental lock-out when an allowlist contains only plugin tools:
```typescript
function stripPluginOnlyAllowlist(policy, groups, coreTools): AllowlistResolution {
  // Returns { strippedAllowlist: true } when policy is plugin-only
  // Prevents disabling all core tools by misconfigured plugin policies
}
```

**Why?** Tool provenance tracking (core vs plugin, author, deprecation status) makes the tool system auditable. The gateway protocol enables dynamic rendering without embedded registries.

### PDF Tool with Native Provider Support (NEW)

A new PDF analysis tool supports both native document-aware and fallback extraction paths:

```typescript
// Native path: Direct API calls for Anthropic/Google
// - Anthropic: sends raw PDF via DocumentBlockParam
// - Google Gemini: sends PDF bytes via inlineData with application/pdf MIME

// Fallback path: For providers without native PDF support
// - Extract text via pdfjs-dist
// - Rasterize pages to images via @napi-rs/canvas
// - Send through vision/text completion path

// Config:
agents.defaults.pdfModel = { primary, fallbacks }  // defaults to imageModel
agents.defaults.pdfMaxBytesMb = 10
agents.defaults.pdfMaxPages = 20
```

**Why?** Adds document understanding without requiring model-specific integrations; leverages existing vision infrastructure with graceful fallback to text extraction.

### Tool Name Normalization Hardening (NEW)

Tool name validation now includes consistent whitespace trimming, alias resolution, and validation against persisted tool-call names:

```typescript
function normalizeToolName(name: string) {
  const normalized = name.trim().toLowerCase();
  return TOOL_NAME_ALIASES[normalized] ?? normalized;
}
```

**Why?** Prevents manipulation via special characters or whitespace-padded tool names in tool-call transcripts.

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

#### 4. Centralized Fallback Resolution (NEW)

Fallback logic was unified into a single resolution engine with explicit cooldown/auth handling:

```typescript
function resolveFallbackCandidates(params: {
  cfg: OpenClawConfig;
  provider: string;
  model: string;
  fallbacksOverride?: string[];  // Per-agent or per-job override
}): ModelCandidate[]
// 1. Normalize primary to configured defaults
// 2. Resolve configured fallback chain or explicit override
// 3. Deduplicate by modelKey(provider, model)
// 4. Skip allowlist enforcement (fallbacks are explicit user intent)
```

**Cooldown decision logic:**
```typescript
type CooldownDecision =
  | { type: "skip"; reason: FailoverReason; error: string }
  | { type: "attempt"; reason: FailoverReason; markProbe: boolean }

// Key behaviors:
// - Primary model retried periodically during cooldown (30s min, configurable)
// - Entire provider skipped if auth/billing issues on all profiles
// - Same-provider rate-limit failures can fall back to sibling models
// - Per-agent fallbacksOverride replaces global config
```

**Default fallback:** Agents without explicit model config now fall back to `agents.defaults.model`, ensuring unspecified agents don't fail immediately.

**Why centralized?** Previously, fallback logic was scattered across multiple call sites. Centralization ensures consistent cooldown awareness, probe throttling, and override support.

#### 5. Adaptive Thinking for Claude 4.6 (NEW)

A new thinking mode supports Anthropic's recommended dynamic allocation:

```typescript
type ThinkLevel =
  | "off" | "minimal" | "low" | "medium" | "high" | "xhigh"
  | "adaptive";  // NEW

agents.defaults.thinkingDefault = "adaptive"  // Claude 4.6 default
```

**Provider mappings:**
- Anthropic: `{ thinking: { type: "adaptive" }, output_config: { effort: "medium" } }`
- OpenRouter: `"adaptive"` → `reasoning.effort: "medium"`
- Google: `"adaptive"` → `thinkingLevel: "MEDIUM"`

**Unified thinking precedence:**
1. Per-request override (user command)
2. Per-agent config
3. Per-model defaults (model catalog)
4. Global `agents.defaults.thinkingDefault`

**Why?** Adaptive mode avoids the deprecated `budget_tokens` field on newer models. It provides a single setting that works across all supported providers.

#### 6. Reasoning Preservation in Fallback (NEW)

Thinking/reasoning configuration is now mapped through the failover chain:

```typescript
// Problem: Fallback to different provider loses thinking payload structure
// Solution: Map reasoning config through failover to maintain requested thinking levels
```

**Why?** Extended thinking (Claude, DeepSeek, etc.) should remain available even when the primary model fails over. Without reasoning preservation, a fallback would silently drop the user's requested thinking level.

#### 7. OpenRouter Reasoning Injection Guards (NEW)

Certain models don't support reasoning.effort injection:

```typescript
function isOpenRouterReasoningUnsupported(modelId: string): boolean {
  return modelId.toLowerCase().startsWith("x-ai/");  // Grok models
}

// Also: openrouter/auto routing model skips reasoning injection
const skipReasoningInjection = modelId === "auto" || isOpenRouterReasoningUnsupported(modelId);
```

**Why?** Prevents 400-level API errors on models with incompatible reasoning requirements.

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

### Compaction Enhancements (NEW)

Three significant compaction improvements were added:

#### Identifier Preservation in Summaries

Compaction summaries previously lost/mangled opaque identifiers (IDs, hashes, URLs). A new policy ensures they survive:

```typescript
const IDENTIFIER_PRESERVATION_INSTRUCTIONS =
  "Preserve all opaque identifiers exactly as written (no shortening or reconstruction), " +
  "including UUIDs, hashes, IDs, tokens, API keys, hostnames, IPs, ports, URLs, and file names.";

// Config:
agents.defaults.compaction.identifierPolicy = "strict" | "off" | "custom"  // default: "strict"
agents.defaults.compaction.identifierInstructions?: string  // custom instructions
```

**Why?** Without identifier preservation, summaries lose references to resources, sessions, and users — making them useless for follow-up actions.

#### Tool Result Details Exclusion

Tool result `.details` payloads are stripped from token accounting:

```typescript
function estimateMessagesTokens(messages: AgentMessage[]): number {
  // SECURITY: toolResult.details can contain untrusted/verbose payloads
  const safe = stripToolResultDetails(messages);
  return safe.reduce((sum, msg) => sum + estimateTokens(msg), 0);
}
```

**Why?** Adversarial tool outputs (huge responses) could inflate token counts and trigger premature compaction. Details also never reach the LLM during compaction via separate sanitization.

#### OpenAI Responses Server-Side Compaction

Auto-enabled for OpenAI Responses API models:

```typescript
function shouldEnableOpenAIResponsesServerCompaction(model, extraParams): boolean {
  if (configured === false) return false;          // Explicit opt-out
  if (!shouldForceResponsesStore(model)) return false;
  if (configured === true) return true;            // Explicit opt-in
  return model.provider === "openai";              // Auto-enable for direct OpenAI
}
```

**Why?** Offloads compaction work from the gateway to OpenAI's infrastructure, reducing compute overhead.

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

### ACP Thread-Bound Agents (NEW)

Anthropic Cloud Platform (ACP) support enables agents bound to platform threads (e.g., Discord threads):

- **ACP Runtime Registration:** Plugin backend infrastructure for `acpx` runtime
- **Unified Dispatch:** ACP sessions routed through core dispatch + lifecycle cleanup
- **Target Validation:** Require explicit ACP target for runtime spawns (prevents cross-tenant leaks)
- **Metadata Seeding:** Persisted metadata for all ACP spawns

**Why?** Thread-bound agents provide conversation-scoped instances without session proliferation, with improved isolation and cleanup semantics.

### sessions_spawn Sandbox Require Mode (NEW)

Sub-agent spawning now supports a `sandbox` parameter:

```typescript
const SESSIONS_SPAWN_SANDBOX_MODES = ["inherit", "require"] as const;

// "inherit": use parent's sandbox mode (default, backward-compatible)
// "require": fail if parent is not in sandbox (enforces sandboxing)
```

**Why?** Prevents privilege escalation — a sandboxed parent can enforce that all children also run sandboxed.

---

## New Provider Integrations (NEW)

### Volcengine/Byteplus Coding-Plan Auth Scoping

Coding-plan model variants (`volcengine-plan`, `byteplus-plan`) use different provider IDs than their auth credentials. A scoped normalization ensures they leverage the base provider's stored credentials:

```typescript
function normalizeProviderIdForAuth(provider: string): string {
  if (provider === "volcengine-plan") return "volcengine";
  if (provider === "byteplus-plan") return "byteplus";
  return provider;
}
```

### Kimi Web Search

Moonshot Kimi added as a fourth web search provider with auto-detection:

```typescript
// Auto-detection priority: Brave → Gemini → Perplexity → Grok → Kimi
tools.webSearch.kimi.apiKey = ${KIMI_API_KEY}
```

### Gemini Google Search Grounding

Gemini can now use Google Search grounding to fetch web results:

```typescript
// Uses Gemini's tools API with Google Search tool
// Resolves grounding redirect URLs via parallel HEAD requests (5s timeout)
// Returns citations alongside results
agents.defaults.tools.webSearch.gemini.model = "gemini-2.5-flash"  // default
```

**Why?** Enables free/cheap web search via Gemini's Google integration — no separate search service subscription needed.

### OpenRouter "Any Model ID" Pass-Through

OpenRouter no longer restricted to a hardcoded prefix list. Any valid OpenRouter model ID is now accepted:

```typescript
// If model not found in local catalog, create on-the-fly with conservative defaults
if (provider === "openrouter" && !found) {
  return { provider: "openrouter", id: modelId, contextWindow: 32_000, maxTokens: 4_000 };
}
```

**Why?** Users can immediately use newly-released OpenRouter models without waiting for catalog updates.

### OpenRouter Caching & Attribution

- **System message caching:** Injects `cache_control: { type: "ephemeral" }` on system prompts for Anthropic models on OpenRouter
- **App attribution:** Sends `HTTP-Referer: https://openclaw.ai` and `X-Title: OpenClaw` headers

### Z.AI Tool Stream

Z.AI provider supports `tool_stream: true` for real-time tool call streaming (enabled by default).

### Google Thinking Payload Sanitization

Guards against invalid negative `thinkingBudget` values that `pi-ai` emits for some Gemini 3.1 model IDs. Negative budgets are silently removed before API calls.

---

## Security Hardening (NEW)

### Docker Browser Container Chromium Flags

Hardened browser sandbox environment with strict Chromium flags — disables plugins, extensions, and enforces sandboxing with restricted feature policies.

### Private-Network Web Search Citation Blocking

Prevents the `web_search` tool from returning results pointing to private network addresses:

```
Blocked: 127.0.0.1, localhost, 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, IPv6 private ranges
```

**Why?** Prevents SSRF escalation via search result citations that redirect to internal services.

### Workspace Bootstrap Boundary Enforcement

All sandbox workspace seeding now uses `openBoundaryFile()` guards (covered in Bootstrap Boundary Hardening above).

---

## StreamFn Wrapper Pipeline (NEW)

Provider-specific payload mutations are composed via higher-order streaming function wrappers in `extra-params.ts`:

```
agent.streamFn
  ├── createStreamFnWithExtraParams       — Temperature, maxTokens, transport
  ├── createAnthropicBetaHeadersWrapper   — Beta headers (1M context, OAuth)
  ├── createOpenRouterWrapper             — App headers, reasoning.effort
  ├── createOpenRouterSystemCacheWrapper  — System message cache_control
  ├── createGoogleThinkingPayloadWrapper  — Gemini thinking config sanitization
  ├── createOpenAIResponsesContextManagementWrapper  — Server-side compaction
  ├── createZaiToolStreamWrapper          — tool_stream parameter
  └── createBedrockNoCacheWrapper         — Disable cache for non-Anthropic Bedrock
```

**Why?** Clean separation of provider-specific concerns. Each wrapper is independently testable and composable. Adding a new provider requires only a new wrapper, not changes to the core pipeline.

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

### 8. Composable Provider Wrappers (NEW)
StreamFn wrappers chain provider-specific payload mutations without coupling providers to each other. Each wrapper is independently testable.

### 9. Centralized Fallback with Cooldown Awareness (NEW)
Unified fallback resolution with probe throttling, provider-level auth awareness, and per-agent overrides ensures reliability without cascading failures.

### 10. Identifier-Preserving Compaction (NEW)
Summaries preserve opaque identifiers (UUIDs, URLs, hashes) so compacted context remains actionable for follow-up operations.

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
| Data-driven tool catalog | Dynamic tool registry exposed via API for all UIs |
| Centralized model fallback | Reliable multi-provider orchestration with cooldown awareness |
| Adaptive thinking | Cost-optimized reasoning that adapts to task complexity |
| Identifier-preserving compaction | Summaries remain actionable for project reference tracking |
| StreamFn wrapper pipeline | Composable provider integrations without coupling |
| Sandbox require mode | Enforce security boundaries in delegated agent chains |

The key difference: OpenClaw's agents serve *one person across many channels*. Our personal agent network has *many agents (one per employee) collaborating across a shared hub*.
