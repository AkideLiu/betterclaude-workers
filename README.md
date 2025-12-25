# BetterClaude Gateway

An intelligent Claude API proxy built on Cloudflare Workers that automatically fixes orphaned tool_result errors.

## Features

- **Auto Error Fix**: Automatically detects and removes orphaned `tool_result` blocks that cause 400 errors
- **Proactive Cleanup**: Cleans messages before API calls to prevent errors
- **Smart Retry**: Falls back to reactive cleanup if proactive detection misses edge cases
- **Transparent Proxy**: Preserves all headers and client information
- **Edge Computing**: Runs on Cloudflare Workers for low latency worldwide

## The Problem

When using Claude with tools, the message history can become corrupted with orphaned `tool_result` blocks - results that reference `tool_use` calls that no longer exist in the conversation. This causes Claude API to return 400 errors:

```
tool_result block(s) that reference non-existent tool_use ids
```

BetterClaude automatically detects and removes these orphaned blocks, allowing the conversation to continue.

## How It Works

1. **Proactive Detection**: Before making the API call, scans messages for orphaned `tool_result` blocks and removes them
2. **API Call**: Forwards the cleaned request to the target Claude API
3. **Reactive Fallback**: If a 400 error still occurs, parses the error to identify remaining orphans and retries once

### Architecture

![BetterClaude Architecture](https://net-oss.akclau.de/static/akide/uPic/arch-20251224yFxg5N.png)

## Usage

Prefix your Claude API endpoint with the gateway URL:

```
https://<YOUR_DOMAIN>/claude/<TARGET_HOST>/v1/messages
```

### Examples

**Direct Anthropic API:**
```
https://api.anthropic.com/v1/messages
→ https://<YOUR_DOMAIN>/claude/api.anthropic.com/v1/messages
```

**Third-party Claude API providers:**
```
https://some-provider.com/v1/messages
→ https://<YOUR_DOMAIN>/claude/some-provider.com/v1/messages
```

## Deployment

### Prerequisites

- [Node.js](https://nodejs.org/) (v20+)
- [Cloudflare account](https://dash.cloudflare.com/)
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/)

### Setup

1. Clone and install dependencies:
   ```bash
   cd better_claude
   npm install
   ```

2. Configure `wrangler.jsonc`:
   - Set your worker name
   - Add your domain routes

3. Deploy:
   ```bash
   npm run deploy
   ```

### Development

```bash
npm run dev    # Start local dev server at http://localhost:8787/
```

## Configuration

### wrangler.jsonc

```jsonc
{
  "name": "your-worker-name",
  "main": "src/index.ts",
  "compatibility_date": "2025-12-13",
  "routes": [
    {
      "pattern": "<YOUR_DOMAIN>/*",
      "zone_name": "<YOUR_ZONE>"
    }
  ]
}
```

## Project Structure

```
better_claude/
├── src/
│   ├── index.ts              # Main worker entry point
│   ├── router.ts             # URL routing logic
│   ├── proxy.ts              # Request proxying with retry
│   ├── retry-handler.ts      # Retry logic with cleanup
│   ├── proactive-cleanup.ts  # Orphan detection algorithm
│   ├── error-detector.ts     # Error parsing utilities
│   ├── streaming-handler.ts  # SSE stream handling
│   └── env.d.ts              # Environment type definitions
├── wrangler.jsonc            # Cloudflare Worker configuration
├── tsconfig.json             # TypeScript configuration
└── package.json              # Dependencies
```

## API Endpoints

| Endpoint | Description |
|----------|-------------|
| `/` | Info endpoint |
| `/health` | Health check |
| `/claude/{host}/{path}` | Proxy to Claude API |

## How the Cleanup Works

The orphan detection algorithm:

1. **Build tool_use index**: Scans all messages to find all `tool_use` blocks and their IDs
2. **Find orphans**: Identifies `tool_result` blocks that reference non-existent `tool_use` IDs
3. **Remove orphans**: Filters out orphaned `tool_result` blocks from messages
4. **Clean empty messages**: Removes user messages that become empty after cleanup

## License

MIT
