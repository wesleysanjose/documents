# Design Doc 11: Automation System

## Overview

The automation system enables agents to operate without real-time user input via four mechanisms: **heartbeat** (periodic LLM invocation for maintenance tasks), **cron** (scheduled tasks with time expressions), **polling** (event loop that checks external sources), and **webhooks** (HTTP endpoints that trigger agent turns). Together they make agents proactive rather than purely reactive.

## Core Concept

An automated turn is identical to a user-initiated turn from the agent's perspective — it goes through the same prompt builder, tool pipeline, and model call. The only difference is the input source: instead of a user message, the system constructs the message from the automation spec. This means all existing policy, memory, and routing apply unchanged.

---

## Data Model

```typescript
interface AutomationSpec {
  id: string;
  agentId: string;
  sessionKey: string;
  type: AutomationType;
  enabled: boolean;

  // Type-specific config
  heartbeat?: HeartbeatConfig;
  cron?: CronConfig;
  poll?: PollConfig;
  webhook?: WebhookConfig;

  // Common constraints
  maxTokens?: number;
  toolProfile?: string;
  isOwner?: boolean;
  metadata?: Record<string, unknown>;
}

type AutomationType = "heartbeat" | "cron" | "poll" | "webhook";

interface HeartbeatConfig {
  intervalMs: number;           // how often to fire (default: 15 min)
  promptFile: string;           // file to read for the heartbeat prompt (HEARTBEAT.md)
  skipIfActive?: boolean;       // skip if agent was recently active
  activeThresholdMs?: number;   // "recently active" window (default: 5 min)
}

interface CronConfig {
  expression: string;           // cron expression: "0 9 * * MON-FRI"
  payload: string;              // message/prompt to send
  timezone?: string;            // IANA timezone for cron evaluation
  jitterMs?: number;            // random delay to prevent thundering herd
}

interface PollConfig {
  intervalMs: number;           // check interval
  source: PollSource;
  filter?: PollFilter;
  payloadTemplate: string;      // message template with {{data}} placeholder
  dedupe?: boolean;             // skip if same data as last poll
}

interface WebhookConfig {
  path: string;                 // URL path: "/webhooks/agent/my-event"
  secret?: string;              // HMAC secret for signature verification
  payloadTemplate: string;      // message template
  method?: "GET" | "POST";
  allowedIPs?: string[];        // source IP allowlist
}

// Poll sources
type PollSource =
  | { type: "http"; url: string; headers?: Record<string, string> }
  | { type: "file"; path: string }
  | { type: "command"; command: string; args?: string[] }
  | { type: "channel_summary" };  // summarize recent channel messages
```

---

## Heartbeat System

The heartbeat fires on an interval, reads `HEARTBEAT.md` from the workspace, and sends it as the user message:

```typescript
class HeartbeatScheduler {
  private timers: Map<string, NodeJS.Timeout> = new Map();

  start(spec: AutomationSpec): void {
    if (spec.type !== "heartbeat" || !spec.heartbeat) return;
    if (this.timers.has(spec.id)) return; // already running

    const { intervalMs, promptFile, skipIfActive, activeThresholdMs = 5 * 60_000 } = spec.heartbeat;

    const tick = async () => {
      try {
        // Skip if agent was recently active (user is still talking)
        if (skipIfActive) {
          const lastActive = await getLastActiveTime(spec.agentId, spec.sessionKey);
          if (Date.now() - lastActive < activeThresholdMs) {
            log.debug(`Heartbeat skipped (agent active): ${spec.agentId}`);
            return;
          }
        }

        await this.fireHeartbeat(spec);
      } catch (err) {
        log.warn(`Heartbeat error for ${spec.agentId}: ${err}`);
      }
    };

    const timer = setInterval(tick, intervalMs);
    this.timers.set(spec.id, timer);
    log.info(`Heartbeat started for agent ${spec.agentId} every ${intervalMs}ms`);
  }

  stop(specId: string): void {
    const timer = this.timers.get(specId);
    if (timer) {
      clearInterval(timer);
      this.timers.delete(specId);
    }
  }

  private async fireHeartbeat(spec: AutomationSpec): Promise<void> {
    const promptFile = spec.heartbeat!.promptFile;
    const promptPath = path.join(spec.metadata?.workspaceDir as string, promptFile);

    let prompt: string;
    try {
      prompt = fs.readFileSync(promptPath, "utf8").trim();
    } catch {
      prompt = `Perform routine heartbeat maintenance. Current time: ${new Date().toISOString()}`;
    }

    await dispatchAutomatedTurn({
      agentId: spec.agentId,
      sessionKey: spec.sessionKey,
      message: prompt,
      isHeartbeat: true,
      toolProfile: spec.toolProfile ?? "heartbeat",
      maxTokens: spec.maxTokens ?? 2000,
    });
  }
}
```

### HEARTBEAT.md Format

```markdown
# Heartbeat Instructions

You are running a scheduled heartbeat turn. Perform the following maintenance tasks:

1. Check if there are any pending tasks in TASKS.md and update their status
2. Review recent system events and note anything requiring attention
3. If today is Monday, generate a weekly summary
4. Check memory for items that need follow-up today

Do not request user input. Complete these tasks autonomously and report briefly.
```

---

## Cron Scheduler

```typescript
class CronScheduler {
  private jobs: Map<string, CronJob> = new Map();

  schedule(spec: AutomationSpec): void {
    if (spec.type !== "cron" || !spec.cron) return;
    if (this.jobs.has(spec.id)) this.unschedule(spec.id);

    const { expression, payload, timezone, jitterMs } = spec.cron;

    const job = new CronJob(expression, async () => {
      // Optional jitter to avoid all agents firing simultaneously
      if (jitterMs && jitterMs > 0) {
        await sleep(Math.random() * jitterMs);
      }

      await dispatchAutomatedTurn({
        agentId: spec.agentId,
        sessionKey: spec.sessionKey,
        message: interpolateTemplate(payload, {
          now: new Date().toISOString(),
          timestamp: Date.now(),
        }),
        isHeartbeat: false,
        toolProfile: spec.toolProfile ?? "full",
        maxTokens: spec.maxTokens,
      });
    }, null, true, timezone);

    this.jobs.set(spec.id, job);
  }

  unschedule(specId: string): void {
    const job = this.jobs.get(specId);
    if (job) {
      job.stop();
      this.jobs.delete(specId);
    }
  }
}

function interpolateTemplate(template: string, vars: Record<string, unknown>): string {
  return template.replace(/\{\{(\w+)\}\}/g, (_, key) => String(vars[key] ?? ""));
}
```

---

## Polling

```typescript
class PollScheduler {
  private timers: Map<string, NodeJS.Timeout> = new Map();
  private lastSeen: Map<string, string> = new Map(); // for dedup

  start(spec: AutomationSpec): void {
    if (spec.type !== "poll" || !spec.poll) return;

    const { intervalMs } = spec.poll;
    const timer = setInterval(() => this.tick(spec), intervalMs);
    this.timers.set(spec.id, timer);
  }

  stop(specId: string): void {
    const timer = this.timers.get(specId);
    if (timer) {
      clearInterval(timer);
      this.timers.delete(specId);
    }
  }

  private async tick(spec: AutomationSpec): Promise<void> {
    const { source, filter, payloadTemplate, dedupe } = spec.poll!;

    let rawData: string;
    try {
      rawData = await this.fetchSource(source);
    } catch (err) {
      log.warn(`Poll source error (${spec.id}): ${err}`);
      return;
    }

    // Apply filter
    if (filter && !applyPollFilter(rawData, filter)) return;

    // Deduplicate
    if (dedupe) {
      const fingerprint = hash(rawData);
      if (this.lastSeen.get(spec.id) === fingerprint) return;
      this.lastSeen.set(spec.id, fingerprint);
    }

    const message = interpolateTemplate(payloadTemplate, {
      data: rawData,
      now: new Date().toISOString(),
    });

    await dispatchAutomatedTurn({
      agentId: spec.agentId,
      sessionKey: spec.sessionKey,
      message,
      toolProfile: spec.toolProfile ?? "full",
      maxTokens: spec.maxTokens,
    });
  }

  private async fetchSource(source: PollConfig["source"]): Promise<string> {
    switch (source.type) {
      case "http": {
        const resp = await fetch(source.url, { headers: source.headers });
        return resp.text();
      }
      case "file":
        return fs.readFileSync(source.path, "utf8");
      case "command": {
        const result = execSync(`${source.command} ${(source.args ?? []).join(" ")}`, {
          encoding: "utf8",
          timeout: 10_000,
        });
        return result;
      }
      case "channel_summary":
        return await buildChannelSummary(/* cfg */);
    }
  }
}

interface PollFilter {
  contains?: string;            // only fire if data contains this string
  regex?: string;               // only fire if data matches this regex
  minLength?: number;           // only fire if data has at least N chars
  jsonPath?: string;            // only fire if this JSONPath has a value
}

function applyPollFilter(data: string, filter: PollFilter): boolean {
  if (filter.minLength !== undefined && data.length < filter.minLength) return false;
  if (filter.contains && !data.includes(filter.contains)) return false;
  if (filter.regex && !new RegExp(filter.regex).test(data)) return false;
  return true;
}
```

---

## Webhook Handler

```typescript
class WebhookServer {
  private handlers: Map<string, AutomationSpec> = new Map();

  register(spec: AutomationSpec): void {
    if (spec.type !== "webhook" || !spec.webhook) return;
    this.handlers.set(spec.webhook.path, spec);
  }

  unregister(spec: AutomationSpec): void {
    if (spec.webhook?.path) {
      this.handlers.delete(spec.webhook.path);
    }
  }

  async handleRequest(req: IncomingMessage, res: ServerResponse): Promise<void> {
    const url = new URL(req.url ?? "/", "http://localhost");
    const spec = this.handlers.get(url.pathname);

    if (!spec || !spec.webhook) {
      res.writeHead(404);
      res.end("Not found");
      return;
    }

    const { secret, payloadTemplate, allowedIPs } = spec.webhook;

    // IP allowlist
    if (allowedIPs && !allowedIPs.includes(getClientIP(req))) {
      res.writeHead(403);
      res.end("Forbidden");
      return;
    }

    // Read body
    const body = await readBody(req);

    // Signature verification
    if (secret) {
      const sig = req.headers["x-signature"] as string;
      if (!verifyHmac(body, secret, sig)) {
        res.writeHead(401);
        res.end("Invalid signature");
        return;
      }
    }

    // Parse and template
    let parsed: unknown;
    try {
      parsed = JSON.parse(body);
    } catch {
      parsed = body;
    }

    const message = interpolateTemplate(payloadTemplate, {
      data: typeof parsed === "string" ? parsed : JSON.stringify(parsed, null, 2),
      now: new Date().toISOString(),
    });

    // Respond immediately (don't wait for agent)
    res.writeHead(202, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ accepted: true }));

    // Fire agent turn asynchronously
    setImmediate(async () => {
      await dispatchAutomatedTurn({
        agentId: spec.agentId,
        sessionKey: spec.sessionKey,
        message,
        toolProfile: spec.toolProfile ?? "full",
        maxTokens: spec.maxTokens,
        isOwner: spec.isOwner,
      });
    });
  }
}

function verifyHmac(body: string, secret: string, signature: string): boolean {
  const expected = crypto
    .createHmac("sha256", secret)
    .update(body)
    .digest("hex");
  return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
}
```

---

## Automated Turn Dispatch

All automation types funnel through a single dispatch function:

```typescript
interface AutomatedTurnParams {
  agentId: string;
  sessionKey: string;
  message: string;
  isHeartbeat?: boolean;
  toolProfile?: string;
  maxTokens?: number;
  isOwner?: boolean;
}

async function dispatchAutomatedTurn(params: AutomatedTurnParams): Promise<void> {
  const { agentId, sessionKey, message, isHeartbeat = false } = params;

  // Emit as a system event (appears in next turn's system prompt)
  emitSystemEvent(sessionKey, {
    text: `Automated turn: ${isHeartbeat ? "heartbeat" : "scheduled"}`,
    ts: Date.now(),
  });

  // Construct synthetic channel message
  const syntheticMsg: ChannelMessage = {
    channelId: `automation:${agentId}`,
    channelType: "cli",
    senderId: "system",
    content: { text: message },
    receivedAt: Date.now(),
    metadata: { isAutomated: true, isHeartbeat },
  };

  // Get agent context
  const agent = await agentPool.getOrCreate(agentId, sessionKey);

  // Dispatch through normal pipeline with automation-specific overrides
  await agent.handleMessage({
    message: syntheticMsg,
    binding: {
      agentId,
      sessionKey,
      channelId: `automation:${agentId}`,
      channelType: "cli",
      tier: "exact_channel",
      config: {
        toolProfile: params.toolProfile,
        isOwner: params.isOwner ?? false,
      },
    },
    sessionKey,
    isOwner: params.isOwner ?? false,
    maxTokens: params.maxTokens,
    isHeartbeat,
  });
}
```

---

## System Events Integration

Automation turns emit system events visible in the next human turn:

```typescript
function emitSystemEvent(sessionKey: string, event: { text: string; ts: number }): void {
  // System events are drained at the start of the next turn
  // and injected as "System: ..." lines in the message
  getSystemEventQueue(sessionKey).push(event);
}
```

---

## Config

```yaml
agents:
  main:
    automation:
      heartbeat:
        enabled: true
        intervalMs: 900000        # 15 minutes
        promptFile: HEARTBEAT.md
        skipIfActive: true
        activeThresholdMs: 300000 # 5 minutes

      cron:
        - id: morning-briefing
          expression: "0 9 * * MON-FRI"
          payload: "Good morning! Please check TASKS.md and send me a briefing."
          timezone: America/New_York

        - id: weekly-summary
          expression: "0 17 * * FRI"
          payload: "End of week. Summarize what was accomplished this week."
          timezone: America/New_York

      poll:
        - id: github-notifications
          intervalMs: 300000      # 5 minutes
          source:
            type: http
            url: "https://api.github.com/notifications"
            headers:
              Authorization: "Bearer ${GITHUB_TOKEN}"
          filter:
            minLength: 10
          dedupe: true
          payloadTemplate: "New GitHub notifications:\n\n{{data}}"

      webhooks:
        - id: github-push
          path: /webhooks/github/push
          secret: "${GITHUB_WEBHOOK_SECRET}"
          payloadTemplate: "GitHub push event:\n\n{{data}}\n\nReview and update CHANGELOG if needed."
```

---

## Implementation Checklist

- [ ] `AutomationSpec` with `type`, `heartbeat`, `cron`, `poll`, `webhook`
- [ ] `HeartbeatScheduler` — setInterval, skip-if-active check, HEARTBEAT.md read
- [ ] `CronScheduler` — cron expression parsing, jitter support, timezone
- [ ] `PollScheduler` — setInterval, HTTP/file/command sources, dedup, filter
- [ ] `WebhookServer` — HTTP handler, IP allowlist, HMAC verification, async dispatch
- [ ] `dispatchAutomatedTurn()` — single dispatch entry point for all automation types
- [ ] `emitSystemEvent()` — announce automation turns in next human turn
- [ ] `interpolateTemplate()` — `{{var}}` substitution
- [ ] `applyPollFilter()` — contains, regex, minLength checks
- [ ] `verifyHmac()` — timing-safe comparison with SHA-256
- [ ] `PollFilter` with 4 filter types
- [ ] Heartbeat system prompt section injection (via `isHeartbeat: true` in `PromptBuildContext`)
- [ ] Automation event filtering in `drainFormattedSystemEvents()` (filter heartbeat noise)
- [ ] All automation types pass through same agent pipeline (prompt, policy, model call)
