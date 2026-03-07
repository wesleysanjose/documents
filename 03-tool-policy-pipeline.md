# Design Doc 03: 8-Layer Tool Policy Pipeline

## Overview

Before any tool is offered to the LLM in a `tools` parameter, every registered tool passes through 8 sequential policy layers. Each layer can allow, deny, or transform the tool. Layers run in a fixed order; a tool denied at any layer is removed from the list (not errored). The final list of allowed tools is passed to the API call.

## Core Concept

The LLM never sees the raw tool registry. It only sees what passes all 8 layers. This means:
- Feature flags can hide experimental tools
- Depth limits prevent infinite recursion in sub-agents
- Owner-only tools are hidden from untrusted callers
- Plugin-contributed tools can be allowlisted per-channel
- Hooks let users override policy at the last moment

`builtInTools` is always `[]` — the framework provides no hardcoded built-in tools. Every tool comes from plugins, skills, or user code.

---

## Pipeline Layers

```typescript
type PolicyLayer =
  | "profile_filter"    // L1: apply named tool profile (e.g., "readonly", "full")
  | "group_expansion"   // L2: expand group aliases into individual tool names
  | "depth_restriction" // L3: restrict tools by sub-agent call depth
  | "owner_gate"        // L4: require owner-level auth for privileged tools
  | "channel_allowlist" // L5: per-channel tool allowlist (plugin-contributed tools)
  | "schema_normalize"  // L6: normalize tool schemas (strip invalid fields)
  | "hook_override"     // L7: user-defined hook can add/remove/reorder tools
  | "dedup_validate";   // L8: deduplicate by name, validate final schema
```

---

## Data Model

```typescript
interface AgentTool {
  name: string;
  description: string;
  inputSchema: JSONSchema;
  handler: ToolHandler;
  // Policy metadata
  groups?: string[];          // logical groups: ["filesystem", "read-only"]
  minDepth?: number;          // only available at this call depth or deeper
  maxDepth?: number;          // only available up to this call depth
  ownerOnly?: boolean;        // requires owner-level session
  pluginId?: string;          // which plugin registered this tool
  channelAllowlist?: string[]; // if set, only available on these channel IDs
  tags?: string[];
}

interface ToolPipelineContext {
  tools: AgentTool[];         // mutable — each layer reads and writes this
  profile: ToolProfile;
  callDepth: number;          // 0 = main agent, 1 = first sub-agent, etc.
  isOwner: boolean;           // true if current session has owner auth
  channelId?: string;
  cfg: AgentConfig;
  sessionId: string;
}

type ToolHandler = (input: unknown, ctx: ToolCallContext) => Promise<unknown>;
```

---

## Tool Profiles

```typescript
interface ToolProfile {
  name: string;
  // Explicit include list (takes priority over excludeGroups)
  includeTools?: string[];
  // Groups to exclude
  excludeGroups?: string[];
  // Maximum allowed call depth for any tool in this profile
  maxCallDepth?: number;
}

const BUILTIN_PROFILES: Record<string, ToolProfile> = {
  full: {
    name: "full",
    // No restrictions
  },
  readonly: {
    name: "readonly",
    excludeGroups: ["write", "execute", "destructive"],
  },
  minimal: {
    name: "minimal",
    includeTools: ["memory_search", "web_search"],
  },
  heartbeat: {
    name: "heartbeat",
    excludeGroups: ["user-interactive", "voice"],
    maxCallDepth: 1,
  },
};
```

---

## Layer Implementations

### L1: Profile Filter

```typescript
function applyProfileFilter(ctx: ToolPipelineContext): void {
  const { profile } = ctx;

  if (profile.includeTools) {
    // Whitelist mode: only named tools pass
    const allowed = new Set(profile.includeTools);
    ctx.tools = ctx.tools.filter((t) => allowed.has(t.name));
    return;
  }

  if (profile.excludeGroups) {
    const excluded = new Set(profile.excludeGroups);
    ctx.tools = ctx.tools.filter((t) => {
      return !t.groups?.some((g) => excluded.has(g));
    });
  }
}
```

### L2: Group Expansion

```typescript
// Group aliases expand before depth/auth checks
const GROUP_ALIASES: Record<string, string[]> = {
  "filesystem": ["file_read", "file_write", "file_delete", "file_list"],
  "memory": ["memory_search", "memory_store", "memory_forget"],
  "web": ["web_search", "web_fetch", "web_screenshot"],
  // user-defined groups can be registered via plugin
};

function applyGroupExpansion(ctx: ToolPipelineContext): void {
  // Expand any group-name entries in includeTools
  // (actual tool objects are already in ctx.tools; this is metadata expansion)
  // Nothing to mutate here unless virtual group-tools are registered
}
```

### L3: Depth Restriction

```typescript
function applyDepthRestriction(ctx: ToolPipelineContext): void {
  const { callDepth, profile } = ctx;

  ctx.tools = ctx.tools.filter((t) => {
    // Tool-level depth check
    if (t.minDepth !== undefined && callDepth < t.minDepth) return false;
    if (t.maxDepth !== undefined && callDepth > t.maxDepth) return false;
    // Profile-level depth check
    if (profile.maxCallDepth !== undefined && callDepth > profile.maxCallDepth) {
      // At max depth: only allow leaf tools (tools without sub-call capability)
      return !t.tags?.includes("orchestrator");
    }
    return true;
  });
}
```

### L4: Owner Gate

```typescript
function applyOwnerGate(ctx: ToolPipelineContext): void {
  if (ctx.isOwner) return; // Owner sees all

  ctx.tools = ctx.tools.filter((t) => !t.ownerOnly);
}
```

### L5: Channel Allowlist

```typescript
function applyChannelAllowlist(ctx: ToolPipelineContext): void {
  if (!ctx.channelId) return; // No channel context — skip

  ctx.tools = ctx.tools.filter((t) => {
    // Tools without a channelAllowlist are always visible
    if (!t.channelAllowlist || t.channelAllowlist.length === 0) return true;
    // Plugin-contributed tools are visible only on allowed channels
    return t.channelAllowlist.includes(ctx.channelId!);
  });
}
```

### L6: Schema Normalize

```typescript
// Some models reject schemas with certain fields
const FORBIDDEN_SCHEMA_FIELDS = ["format", "$schema", "$id", "definitions"];

function applySchemaNormalize(ctx: ToolPipelineContext): void {
  ctx.tools = ctx.tools.map((t) => ({
    ...t,
    inputSchema: normalizeSchema(t.inputSchema),
  }));
}

function normalizeSchema(schema: JSONSchema): JSONSchema {
  if (typeof schema !== "object" || !schema) return schema;

  const cleaned: JSONSchema = { ...schema };

  for (const field of FORBIDDEN_SCHEMA_FIELDS) {
    delete cleaned[field];
  }

  // Recursively normalize nested properties
  if (cleaned.properties) {
    cleaned.properties = Object.fromEntries(
      Object.entries(cleaned.properties).map(([k, v]) => [k, normalizeSchema(v)]),
    );
  }

  if (cleaned.items) {
    cleaned.items = normalizeSchema(cleaned.items as JSONSchema);
  }

  return cleaned;
}
```

### L7: Hook Override

```typescript
type ToolPolicyHook = (
  tools: AgentTool[],
  ctx: Omit<ToolPipelineContext, "tools">,
) => AgentTool[] | Promise<AgentTool[]>;

async function applyHookOverride(ctx: ToolPipelineContext): Promise<void> {
  const hooks = ctx.cfg.toolPolicyHooks ?? [];
  for (const hook of hooks) {
    try {
      ctx.tools = await hook(ctx.tools, ctx);
    } catch (err) {
      log.warn(`tool policy hook failed: ${err}`);
      // Non-fatal: continue with current tools
    }
  }
}
```

### L8: Dedup + Validate

```typescript
function applyDedupValidate(ctx: ToolPipelineContext): void {
  const seen = new Set<string>();
  const valid: AgentTool[] = [];

  for (const tool of ctx.tools) {
    if (seen.has(tool.name)) {
      log.warn(`duplicate tool name '${tool.name}' — keeping first`);
      continue;
    }
    if (!isValidToolSchema(tool)) {
      log.warn(`tool '${tool.name}' has invalid schema — excluded`);
      continue;
    }
    seen.add(tool.name);
    valid.push(tool);
  }

  ctx.tools = valid;
}

function isValidToolSchema(tool: AgentTool): boolean {
  return (
    typeof tool.name === "string" &&
    tool.name.length > 0 &&
    tool.name.length <= 64 &&
    /^[a-zA-Z0-9_-]+$/.test(tool.name) &&
    typeof tool.description === "string" &&
    tool.description.length > 0 &&
    typeof tool.inputSchema === "object"
  );
}
```

---

## Pipeline Runner

```typescript
async function runToolPolicyPipeline(
  tools: AgentTool[],
  params: Omit<ToolPipelineContext, "tools">,
): Promise<AgentTool[]> {
  const ctx: ToolPipelineContext = { ...params, tools: [...tools] };

  applyProfileFilter(ctx);          // L1
  applyGroupExpansion(ctx);         // L2
  applyDepthRestriction(ctx);       // L3
  applyOwnerGate(ctx);              // L4
  applyChannelAllowlist(ctx);       // L5
  applySchemaNormalize(ctx);        // L6
  await applyHookOverride(ctx);     // L7
  applyDedupValidate(ctx);          // L8

  return ctx.tools;
}
```

---

## Loop Detection

When a tool handler spawns sub-agent calls, depth increments. Beyond `maxCallDepth`, orchestrator-tagged tools are removed. This prevents runaway recursion:

```typescript
function getSubAgentPipelineContext(
  parentCtx: ToolPipelineContext,
): Omit<ToolPipelineContext, "tools"> {
  return {
    ...parentCtx,
    callDepth: parentCtx.callDepth + 1,
    // Sub-agents default to readonly profile unless explicitly elevated
    profile: BUILTIN_PROFILES.readonly,
  };
}
```

---

## Tool Registration API

```typescript
class ToolRegistry {
  private tools: Map<string, AgentTool> = new Map();

  register(tool: AgentTool): void {
    if (this.tools.has(tool.name)) {
      throw new Error(`Tool '${tool.name}' already registered`);
    }
    this.tools.set(tool.name, tool);
  }

  unregister(name: string): boolean {
    return this.tools.delete(name);
  }

  async resolveForCall(params: Omit<ToolPipelineContext, "tools">): Promise<AgentTool[]> {
    const allTools = [...this.tools.values()];
    return runToolPolicyPipeline(allTools, params);
  }

  getHandler(name: string): ToolHandler | undefined {
    return this.tools.get(name)?.handler;
  }
}
```

---

## Config

```yaml
agents:
  defaults:
    toolProfile: full         # named profile
    maxCallDepth: 3           # global depth limit
    ownerChannels:            # channel IDs that get owner-level access
      - "telegram:12345"
```

---

## Implementation Checklist

- [ ] `AgentTool` interface with `groups`, `minDepth`, `maxDepth`, `ownerOnly`, `channelAllowlist`
- [ ] `ToolPipelineContext` struct
- [ ] `ToolProfile` with `includeTools`, `excludeGroups`, `maxCallDepth`
- [ ] 4 built-in profiles: `full`, `readonly`, `minimal`, `heartbeat`
- [ ] L1 `applyProfileFilter()` — whitelist vs exclude-group modes
- [ ] L2 `applyGroupExpansion()` — group alias map
- [ ] L3 `applyDepthRestriction()` — min/max depth + profile maxCallDepth
- [ ] L4 `applyOwnerGate()` — ownerOnly filter
- [ ] L5 `applyChannelAllowlist()` — per-channel plugin tool visibility
- [ ] L6 `applySchemaNormalize()` — remove `format`, `$schema`, `$id`, `definitions`
- [ ] L7 `applyHookOverride()` — async hooks, non-fatal
- [ ] L8 `applyDedupValidate()` — dedup by name, validate schema shape
- [ ] `runToolPolicyPipeline()` runs all 8 layers in order
- [ ] `ToolRegistry` with `register()`, `unregister()`, `resolveForCall()`
- [ ] `builtInTools` is always `[]` — no hardcoded built-ins
- [ ] Sub-agent context increments `callDepth`, defaults to `readonly` profile
