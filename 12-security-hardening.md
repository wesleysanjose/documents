# Design Doc 12: Security Hardening

## Overview

Security hardening wraps the entire agent pipeline with protections against: SSRF (server-side request forgery), path traversal, prompt injection via untrusted channel content, sandbox escapes, and boundary violations between trust levels. Each protection is a targeted, composable guard — not a monolithic security layer.

## Core Concept

Trust levels:
- **Owner**: full access, same person who configured the agent
- **User**: authenticated channel user, can use agent normally
- **Untrusted**: unauthenticated or from unknown source

The security model is **boundary enforcement**: establish trust at entry points (channel auth, webhook HMAC, pairing confirmation) and propagate trust level through the pipeline. Never re-establish trust mid-pipeline based on content from untrusted sources.

---

## Trust Model

```typescript
type TrustLevel = "owner" | "user" | "untrusted";

interface TrustContext {
  level: TrustLevel;
  senderId: string;
  channelId: string;
  authenticatedAt?: number;
  authMethod?: "pairing" | "allowlist" | "hmac" | "api_key" | "auto";
}

function resolveTrustLevel(params: {
  msg: ChannelMessage;
  binding: ChannelBinding;
  pairingStore: PairingStore;
  cfg: AgentConfig;
}): TrustContext {
  const { msg, binding } = params;

  // Owner: explicitly configured as owner in binding
  if (binding.config.isOwner) {
    return {
      level: "owner",
      senderId: msg.senderId,
      channelId: msg.channelId,
      authMethod: "allowlist",
    };
  }

  // User: confirmed pairing exists
  if (binding.tier === "exact_channel" || binding.tier === "acp_route") {
    return {
      level: "user",
      senderId: msg.senderId,
      channelId: msg.channelId,
      authMethod: "pairing",
    };
  }

  // Untrusted: auto-paired but not confirmed, or pattern-matched
  return {
    level: "untrusted",
    senderId: msg.senderId,
    channelId: msg.channelId,
    authMethod: "auto",
  };
}
```

---

## 1. SSRF Prevention

Any HTTP request made by agent tools must pass through the SSRF guard:

```typescript
const SSRF_BLOCKED_RANGES = [
  // Private IPv4 ranges
  /^10\./,
  /^172\.(1[6-9]|2[0-9]|3[01])\./,
  /^192\.168\./,
  /^127\./,
  /^0\./,
  /^169\.254\./,   // link-local
  /^::1$/,         // IPv6 loopback
  /^fc00:/i,       // IPv6 ULA
  /^fe80:/i,       // IPv6 link-local
];

const SSRF_BLOCKED_HOSTNAMES = new Set([
  "localhost",
  "metadata.google.internal",
  "169.254.169.254",  // AWS/GCP metadata
  "100.100.100.200",  // Alibaba Cloud metadata
]);

interface SSRFCheckResult {
  allowed: boolean;
  reason?: string;
}

async function checkSSRF(url: string, cfg: AgentConfig): Promise<SSRFCheckResult> {
  let parsed: URL;
  try {
    parsed = new URL(url);
  } catch {
    return { allowed: false, reason: "invalid URL" };
  }

  // Protocol check
  const allowedProtocols = cfg.security?.allowedProtocols ?? ["https:", "http:"];
  if (!allowedProtocols.includes(parsed.protocol)) {
    return { allowed: false, reason: `protocol not allowed: ${parsed.protocol}` };
  }

  const hostname = parsed.hostname.toLowerCase();

  // Blocked hostname check
  if (SSRF_BLOCKED_HOSTNAMES.has(hostname)) {
    return { allowed: false, reason: `blocked hostname: ${hostname}` };
  }

  // Resolve hostname to IP and check ranges
  let addresses: string[];
  try {
    addresses = await resolveHostname(hostname);
  } catch {
    return { allowed: false, reason: `hostname resolution failed: ${hostname}` };
  }

  for (const ip of addresses) {
    for (const pattern of SSRF_BLOCKED_RANGES) {
      if (pattern.test(ip)) {
        return { allowed: false, reason: `blocked IP range: ${ip}` };
      }
    }
  }

  // Domain allowlist (if configured)
  if (cfg.security?.domainAllowlist) {
    const allowed = cfg.security.domainAllowlist.some(
      (d) => hostname === d || hostname.endsWith(`.${d}`),
    );
    if (!allowed) {
      return { allowed: false, reason: `domain not in allowlist: ${hostname}` };
    }
  }

  return { allowed: true };
}

// Wrapper for fetch calls in tools
async function secureFetch(
  url: string,
  options: RequestInit,
  cfg: AgentConfig,
): Promise<Response> {
  const check = await checkSSRF(url, cfg);
  if (!check.allowed) {
    throw new Error(`SSRF blocked: ${url} — ${check.reason}`);
  }
  return fetch(url, {
    ...options,
    // Disable redirects to private ranges
    redirect: "manual",
  });
}
```

---

## 2. Path Traversal Guard

All file read/write operations from tool calls must be validated:

```typescript
interface PathCheckResult {
  allowed: boolean;
  reason?: string;
  resolvedPath?: string;
}

function checkPathAccess(params: {
  requestedPath: string;
  allowedRoots: string[];
  operation: "read" | "write" | "delete" | "list";
  cfg: AgentConfig;
}): PathCheckResult {
  const { requestedPath, allowedRoots, operation } = params;

  let resolved: string;
  try {
    resolved = path.resolve(requestedPath);
  } catch {
    return { allowed: false, reason: "invalid path" };
  }

  // Path must be inside at least one allowed root
  const insideAllowed = allowedRoots.some((root) => {
    const resolvedRoot = path.resolve(root);
    return resolved.startsWith(resolvedRoot + path.sep) || resolved === resolvedRoot;
  });

  if (!insideAllowed) {
    return {
      allowed: false,
      reason: `path '${resolved}' is outside allowed roots: ${allowedRoots.join(", ")}`,
    };
  }

  // Symlink resolution: check real path too
  try {
    const realPath = fs.realpathSync(resolved);
    const realInsideAllowed = allowedRoots.some((root) => {
      const realRoot = fs.realpathSync(path.resolve(root));
      return realPath.startsWith(realRoot + path.sep) || realPath === realRoot;
    });

    if (!realInsideAllowed) {
      return {
        allowed: false,
        reason: `symlink '${resolved}' resolves to '${realPath}' which is outside allowed roots`,
      };
    }
  } catch {
    // File doesn't exist yet (write operation) — allow if path check passed
    if (operation === "write") {
      return { allowed: true, resolvedPath: resolved };
    }
    return { allowed: false, reason: "path does not exist" };
  }

  // Write/delete operations may be further restricted
  if (operation === "write" || operation === "delete") {
    const writableRoots = params.cfg.security?.writableRoots;
    if (writableRoots) {
      const writable = writableRoots.some((root) => {
        const resolvedRoot = path.resolve(root);
        return resolved.startsWith(resolvedRoot + path.sep);
      });
      if (!writable) {
        return { allowed: false, reason: `write/delete not allowed at: ${resolved}` };
      }
    }
  }

  return { allowed: true, resolvedPath: resolved };
}
```

---

## 3. Prompt Injection Defense

User-supplied content in channel messages must be sanitized before being embedded in the system prompt or tool inputs:

```typescript
const INJECTION_PATTERNS = [
  // Attempts to inject trusted "System:" prefix
  /^system\s*:/im,
  // XML-like tool injection
  /<tool_use\s/i,
  /<tool_result\s/i,
  // Attempts to close and restart system prompts
  /\[end\s+system\s+prompt\]/i,
  /\[start\s+instructions\]/i,
  // Role injection
  /^(user|assistant|system)\s*:\s*/im,
];

function sanitizeUserContent(content: string): string {
  // Rewrite trusted-looking prefixes to untrusted equivalents
  let sanitized = content;

  // "System:" at line start becomes "System (untrusted):"
  sanitized = sanitized.replace(/^system\s*:/gim, "System (untrusted):");

  // Strip null bytes and control characters (except newlines, tabs)
  sanitized = sanitized.replace(/[\x00-\x08\x0b\x0c\x0e-\x1f\x7f]/g, "");

  return sanitized;
}

function detectInjectionAttempt(content: string): boolean {
  return INJECTION_PATTERNS.some((p) => p.test(content));
}

// Apply to all inbound channel message content
function processInboundMessage(msg: ChannelMessage): ChannelMessage {
  const rawText = typeof msg.content === "string"
    ? msg.content
    : (msg.content as { text?: string }).text ?? "";

  if (detectInjectionAttempt(rawText)) {
    log.warn(`Potential prompt injection detected from ${msg.senderId} on ${msg.channelId}`);
  }

  const sanitizedText = sanitizeUserContent(rawText);

  return {
    ...msg,
    content: typeof msg.content === "string"
      ? sanitizedText
      : { ...msg.content, text: sanitizedText },
  };
}
```

---

## 4. Tool Output Sanitization

Tool results returned to the LLM can also contain injection attempts:

```typescript
function sanitizeToolResult(result: unknown): unknown {
  if (typeof result === "string") {
    return sanitizeUserContent(result);
  }
  if (typeof result === "object" && result !== null) {
    return Object.fromEntries(
      Object.entries(result as Record<string, unknown>).map(([k, v]) => [
        k,
        sanitizeToolResult(v),
      ]),
    );
  }
  return result;
}
```

---

## 5. Boundary Guards

Guards at trust level boundaries prevent privilege escalation:

```typescript
// Prevent untrusted users from triggering owner-only commands
function checkTrustBoundary(
  action: string,
  requiredLevel: TrustLevel,
  actualLevel: TrustLevel,
): void {
  const levels: Record<TrustLevel, number> = { untrusted: 0, user: 1, owner: 2 };

  if (levels[actualLevel] < levels[requiredLevel]) {
    throw new SecurityError(
      `Action '${action}' requires '${requiredLevel}' trust but caller has '${actualLevel}'`,
    );
  }
}

class SecurityError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "SecurityError";
  }
}

// Guard for tool handlers
function requireTrust(level: TrustLevel) {
  return function (handler: ToolHandler): ToolHandler {
    return async (input, ctx) => {
      checkTrustBoundary(ctx.toolName, level, ctx.trustLevel);
      return handler(input, ctx);
    };
  };
}

// Usage:
const adminTool: AgentTool = {
  name: "restart_gateway",
  ownerOnly: true,
  handler: requireTrust("owner")(async (input, ctx) => {
    // ... restart logic
  }),
  // ...
};
```

---

## 6. Sandbox for Tool Execution

Tools that execute user-supplied code run in a restricted process:

```typescript
interface ToolSandboxPolicy {
  allowNetworkAccess: boolean;
  allowFilesystem: boolean;
  allowedPaths?: string[];
  maxRuntimeMs: number;
  maxMemoryMb: number;
  allowedEnvVars: string[];
}

const DEFAULT_SANDBOX_POLICY: ToolSandboxPolicy = {
  allowNetworkAccess: false,
  allowFilesystem: false,
  maxRuntimeMs: 10_000,
  maxMemoryMb: 256,
  allowedEnvVars: ["PATH", "HOME"],
};

async function runToolInSandbox(
  handler: ToolHandler,
  input: unknown,
  ctx: ToolCallContext,
  policy: ToolSandboxPolicy = DEFAULT_SANDBOX_POLICY,
): Promise<unknown> {
  // For high-trust tools: run directly
  if (ctx.trustLevel === "owner" && ctx.tool.ownerOnly) {
    return handler(input, ctx);
  }

  // For untrusted tool inputs: wrap with timeout and resource limits
  const timeoutPromise = new Promise<never>((_, reject) =>
    setTimeout(
      () => reject(new Error(`Tool '${ctx.toolName}' exceeded time limit (${policy.maxRuntimeMs}ms)`)),
      policy.maxRuntimeMs,
    ),
  );

  return Promise.race([handler(input, ctx), timeoutPromise]);
}
```

---

## 7. Rate Limiting and Abuse Prevention

```typescript
class AbuseDetector {
  private recentMessages: Map<string, number[]> = new Map();

  check(senderId: string, channelId: string): { blocked: boolean; reason?: string } {
    const key = `${senderId}:${channelId}`;
    const now = Date.now();
    const window = 60_000; // 1 minute
    const maxMessages = 30; // per minute

    const timestamps = (this.recentMessages.get(key) ?? []).filter(
      (t) => now - t < window,
    );
    timestamps.push(now);
    this.recentMessages.set(key, timestamps);

    if (timestamps.length > maxMessages) {
      return {
        blocked: true,
        reason: `Rate limit: ${timestamps.length} messages in 1 minute (max: ${maxMessages})`,
      };
    }

    return { blocked: false };
  }

  // Detect abnormally long messages (potential context stuffing)
  checkMessageLength(content: string, maxChars: number = 50_000): boolean {
    return content.length <= maxChars;
  }
}
```

---

## 8. Secret Detection in Outputs

Prevent the agent from accidentally leaking secrets in responses:

```typescript
const SECRET_PATTERNS = [
  { name: "AWS key", pattern: /AKIA[0-9A-Z]{16}/ },
  { name: "GitHub token", pattern: /ghp_[a-zA-Z0-9]{36}/ },
  { name: "Anthropic key", pattern: /sk-ant-[a-zA-Z0-9-]{80,}/ },
  { name: "OpenAI key", pattern: /sk-[a-zA-Z0-9]{48,}/ },
  { name: "Private key block", pattern: /-----BEGIN (RSA |EC )?PRIVATE KEY-----/ },
];

interface SecretDetectionResult {
  hasSecrets: boolean;
  detected: Array<{ name: string; position: number }>;
}

function scanForSecrets(text: string): SecretDetectionResult {
  const detected: Array<{ name: string; position: number }> = [];

  for (const { name, pattern } of SECRET_PATTERNS) {
    const match = pattern.exec(text);
    if (match) {
      detected.push({ name, position: match.index });
    }
  }

  return { hasSecrets: detected.length > 0, detected };
}

function redactSecrets(text: string): string {
  let redacted = text;
  for (const { pattern } of SECRET_PATTERNS) {
    redacted = redacted.replace(pattern, "[REDACTED]");
  }
  return redacted;
}
```

---

## 9. Security Config

```yaml
security:
  # SSRF
  domainAllowlist:
    - api.github.com
    - api.slack.com
  allowedProtocols:
    - https:

  # Path access
  writableRoots:
    - /home/user/workspace
    - /tmp/agent-scratch

  # Rate limiting
  rateLimiting:
    defaultRpm: 30
    burstMultiplier: 2

  # Prompt injection
  logInjectionAttempts: true
  blockOnInjection: false    # warn only, don't block

  # Secret scanning in outputs
  scanOutputsForSecrets: true
  redactSecretsInLogs: true

  # Tool sandbox
  toolTimeout: 10000
  maxToolOutputBytes: 102400  # 100KB
```

---

## Implementation Checklist

- [ ] `TrustLevel` enum: owner, user, untrusted
- [ ] `TrustContext` with level, senderId, channelId, authMethod
- [ ] `resolveTrustLevel()` — maps binding tier to trust level
- [ ] `checkSSRF()` — private IP ranges, blocked hostnames, domain allowlist
- [ ] `secureFetch()` — wraps fetch with SSRF check, redirect: manual
- [ ] `checkPathAccess()` — allowed roots, symlink resolution, write restriction
- [ ] `sanitizeUserContent()` — rewrite "System:" → "System (untrusted):", strip control chars
- [ ] `detectInjectionAttempt()` — log injection pattern matches
- [ ] `processInboundMessage()` — sanitize all inbound channel content
- [ ] `sanitizeToolResult()` — sanitize tool outputs before LLM sees them
- [ ] `checkTrustBoundary()` — throw SecurityError on privilege violation
- [ ] `requireTrust()` — decorator for tool handlers
- [ ] `runToolInSandbox()` — timeout enforcement for untrusted tool calls
- [ ] `AbuseDetector` — per-sender rate limiting, message length checks
- [ ] `scanForSecrets()` — detect 5 common secret patterns
- [ ] `redactSecrets()` — replace secrets in log output
- [ ] All SSRF-blocked ranges: 10.x, 172.16-31.x, 192.168.x, 127.x, 169.254.x, IPv6 loopback/ULA
- [ ] Security config schema in `AgentConfig`
