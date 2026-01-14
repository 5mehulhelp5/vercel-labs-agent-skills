---
name: slack-agent
description: This skill helps you build deploy, and run a production-ready Slack bot on Vercel using the AI SDK with proper logging, safeguards, and operational practices.
---

# Slack Agent Builder

## Architecture Overview

The Slack agent follows this event flow:
```
User sends Slack message → events.post.ts (Nitro route) → VercelReceiver (Bolt) → Event Listeners → createTextStream (AI + tools) → client.chatStream → Streamed Reply
```

## Key Files Structure

```
server/
├── api/slack/events.post.ts    # HTTP entry point
├── app.ts                       # Bolt app bootstrap
├── env.ts                       # Zod environment validation
├── listeners/
│   ├── events/                  # app_mention, direct_message, reactions
│   ├── commands/                # Slash commands
│   ├── shortcuts/               # Global/message shortcuts
│   ├── actions/                 # Button clicks, select menus
│   └── views/                   # Modal submissions
├── lib/
│   ├── ai/
│   │   ├── respond-to-message.ts
│   │   ├── tools/               # AI tool definitions
│   │   ├── cost-control.ts      # Token estimation & budget
│   │   └── retry-wrapper.ts     # Exponential backoff
│   └── slack/
│       └── modals.ts            # Shared modal definitions
scripts/
└── dev.tunnel.ts                # ngrok tunnel orchestration
manifest.json                    # Slack app configuration
```

## Core Patterns

### 1. Environment Validation with Zod

Always validate environment variables at startup for fail-fast behavior:

```typescript
// server/env.ts
import { z } from "zod";

export const envSchema = z.object({
  SLACK_SIGNING_SECRET: z.string().min(1, "SLACK_SIGNING_SECRET is required"),
  SLACK_BOT_TOKEN: z.string().min(1, "SLACK_BOT_TOKEN is required"),
  AI_GATEWAY_API_KEY: z.string().optional(),
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
});

const result = envSchema.safeParse(process.env);
if (!result.success) {
  console.error("Invalid environment configuration:", result.error.issues);
  throw new Error("Environment validation failed");
}
export const env = result.data;
```

### 2. Correlation Middleware

Add correlation IDs to all requests for distributed tracing:

```typescript
// server/app.ts
app.use(async ({ context, body, payload, next }) => {
  context.correlation = {
    event_id: body.event_id,
    ts: payload.ts ?? payload.message_ts ?? payload.item?.ts,
    thread_ts: payload.thread_ts
  };
  await next();
});
```

### 3. The 3-Second Ack Rule

Slack enforces a hard 3-second timeout. Always acknowledge immediately:

```typescript
export const commandCallback = async ({ ack, respond, logger, context }) => {
  try {
    // ALWAYS ack first - before any heavy work
    await ack();

    // Heavy processing happens after acknowledgment
    const result = await processRequest();

    await respond({
      text: result,
      response_type: "ephemeral", // or "in_channel"
    });
  } catch (error) {
    logger.error({ ...context.correlation, error });
    await respond({ text: "Something went wrong.", response_type: "ephemeral" });
  }
};
```

### 4. Modal Validation Pattern

For modals, validate BEFORE acknowledging to show inline errors:

```typescript
export const viewCallback = async ({ ack, view, client, logger }) => {
  const values = view.state.values;
  const title = values.title_block.title_input.value;

  // Validate first
  if (title.length < 5) {
    await ack({
      response_action: "errors",
      errors: { title_block: "Title must be at least 5 characters" }
    });
    return;
  }

  // Then ack and process
  await ack();
  // ... process submission
};
```

### 5. AI Tool Definition

Tools enable your bot to take actions, not just talk:

```typescript
// server/lib/ai/tools/react-to-message.ts
import { z } from "zod";

export const reactToMessageTool = {
  description: "Add an emoji reaction to a message",
  inputSchema: z.object({
    emoji: z.string().describe("The emoji to add (without colons)"),
  }),
  execute: async ({ emoji }, { experimental_context }) => {
    const { channel, thread_ts, client } = experimental_context;

    try {
      await client.reactions.add({
        channel,
        timestamp: thread_ts,
        name: emoji,
      });
      return `Added :${emoji}: reaction`;
    } catch (error) {
      if (error.data?.error === "already_reacted") {
        return `Already reacted with :${emoji}:`;
      }
      throw error;
    }
  },
};
```

### 6. System Prompt Best Practices

Structure responses for actionable output:

```typescript
system: `You are a Slack assistant.

RESPONSE RULES:
1. Give direct answers immediately - no "I'd be happy to help"
2. Be concise but complete
3. ALWAYS end with exactly two next steps

Format: [Answer]

**Next steps:**
• [Specific action users can take now]
• [Alternative approach or follow-up]

STATUS UPDATES:
- Call updateAgentStatusTool immediately when receiving a request
- Update before every tool execution
- Include specific counts when available (e.g., "analyzing 23 messages")
- Never allow gaps longer than 2 seconds without feedback`
```

### 7. Retry with Exponential Backoff

Handle rate limits and failures gracefully:

```typescript
// server/lib/ai/retry-wrapper.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  options: { maxAttempts: number; backoffMs: number; maxBackoffMs: number }
): Promise<T> {
  let lastError: Error;
  let delay = options.backoffMs;

  for (let attempt = 1; attempt <= options.maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      if (attempt === options.maxAttempts) break;

      // Don't retry client errors (except rate limits)
      if (error.status >= 400 && error.status < 500 && error.status !== 429) {
        throw error;
      }

      await new Promise(r => setTimeout(r, delay));
      delay = Math.min(delay * 2, options.maxBackoffMs);
    }
  }
  throw lastError;
}
```

### 8. Cost Control

Estimate and reject expensive requests before making API calls:

```typescript
// server/lib/ai/cost-control.ts
const PRICING = {
  "gpt-4o-mini": { input: 0.00015, output: 0.0006 },
  "gpt-3.5-turbo": { input: 0.0005, output: 0.0015 },
  "gpt-4o": { input: 0.0025, output: 0.01 },
};

export function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4); // ~4 chars per token
}

export function estimateCost(inputTokens: number, outputTokens: number, model: string): number {
  const pricing = PRICING[model];
  return (inputTokens / 1000) * pricing.input + (outputTokens / 1000) * pricing.output;
}

export function shouldRejectRequest(estimatedCost: number, maxCost = 0.10): boolean {
  return estimatedCost > maxCost;
}
```

## Manifest Configuration

Key sections in `manifest.json`:

```json
{
  "display_information": {
    "name": "My Slack Agent",
    "description": "AI-powered assistant"
  },
  "features": {
    "app_home": { "home_tab_enabled": true },
    "bot_user": { "display_name": "Agent", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "chat:write",
        "commands",
        "im:history",
        "im:read",
        "im:write"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://your-app.vercel.app/api/slack/events",
      "bot_events": ["app_mention", "message.im", "app_home_opened"]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://your-app.vercel.app/api/slack/events"
    }
  }
}
```

## OAuth Scope Principles

- Start minimal, add as needed
- Every scope should map to a specific feature
- Audit scopes regularly - over-scoped bots are security risks
- Common scopes:
  - `app_mentions:read` - Required for @mentions
  - `chat:write` - Required for all bot responses
  - `commands` - Required for slash commands
  - `im:history`, `im:read`, `im:write` - Required for DMs

## Deployment Checklist

### Local Development
1. Create Slack Developer Sandbox
2. Create app from manifest at api.slack.com/apps/new
3. Set environment variables: `SLACK_BOT_TOKEN`, `SLACK_SIGNING_SECRET`, `NGROK_AUTH_TOKEN`
4. Run `slack login` and `slack app link`
5. Set manifest source to `local` in `.slack/config.json`
6. Run `slack run` and confirm manifest changes

### Production Deployment
1. Deploy with `pnpm dlx vercel --prod`
2. Set environment variables in Vercel dashboard
3. Update manifest URLs to production domain
4. Verify Slack Events URL shows green checkmark
5. Test bot responds to mentions in production

## Structured Logging Schema

```typescript
interface LogEntry {
  correlationId: string;
  operation: 'respondToMessage' | 'toolCall' | 'retry';
  model?: string;
  promptTokens?: number;
  completionTokens?: number;
  retryAttempt?: number;
  rateLimitWaitMs?: number;
  channel?: string;
  thread_ts?: string;
  latencyMs?: number;
  error?: string;
}

// Usage with Bolt's Pino logger
logger.info({
  ...context.correlation,
  operation: 'respondToMessage',
  model: 'gpt-4o-mini',
  promptTokens: 150,
  completionTokens: 89,
  latencyMs: 1234
}, "AI response completed");
```

## Service Level Objectives (SLOs)

- Event acknowledgment: 99% within 3 seconds
- Response time (p95): 95% within 15 seconds
- Error rate: <1%
- Availability: 99.9%

## Troubleshooting

### Bot not responding
1. Check health endpoint returns 200
2. Verify Slack Events URL is verified (green checkmark)
3. Confirm bot is invited to the channel
4. Check logs for correlation IDs

### Rate limits (429)
1. Search logs for `rateLimitWaitMs`
2. Verify retry logic is engaging
3. Implement request queuing if needed

### Timeout errors
1. Ensure `ack()` is called before heavy processing
2. Check external API status
3. Examine full request trace via correlation ID

### Modal/Shortcut failures
1. Verify `trigger_id` is used within 3 seconds
2. Check `callback_id`, `block_id`, `action_id` alignment
3. Ensure bot has required scopes

## Key Resources

- Template repo: Deploy with Vercel button creates linked GitHub repo
- Slack API docs: api.slack.com
- Bolt.js docs: slack.dev/bolt-js
- AI SDK docs: sdk.vercel.ai
