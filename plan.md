# Plan: Add WebSocket Endpoint and Headers Support

## Status: ✅ COMPLETED

## Overview

Add support for connecting to Chrome via WebSocket endpoint with custom headers, in addition to the existing `--browserUrl` argument.

## Puppeteer Support Verification ✅

Based on Puppeteer documentation, the `puppeteer.connect()` method supports:

- **`browserWSEndpoint`** (string): WebSocket URL to connect directly to the browser (e.g., `ws://127.0.0.1:9222/devtools/browser/<id>`)
- **`headers`** (Record<string, string>): Custom headers for the WebSocket connection (Node.js only)

Source: `ConnectOptions` interface in Puppeteer Core

## Current Implementation

- `--browserUrl` / `-u`: Connects using `puppeteer.connect({ browserURL: ... })`
- Uses HTTP endpoint which Puppeteer converts to WebSocket internally
- No support for custom headers

## Proposed Changes

### 1. CLI Arguments (src/cli.ts)

Add two new arguments:

```typescript
browserWsEndpoint: {
  type: 'string',
  description: 'WebSocket endpoint to connect to a running Chrome instance (e.g., ws://127.0.0.1:9222/devtools/browser/<id>). Alternative to --browserUrl.',
  alias: 'w',
  conflicts: 'browserUrl',
}

wsHeaders: {
  type: 'string',
  description: 'Custom headers for WebSocket connection in JSON format (e.g., \'{"Authorization":"Bearer token"}\'). Only works with --browserWsEndpoint.',
  coerce: (val: string | undefined) => {
    if (!val) return undefined;
    try {
      const parsed = JSON.parse(val);
      if (typeof parsed !== 'object' || Array.isArray(parsed)) {
        throw new Error('Headers must be a JSON object');
      }
      return parsed;
    } catch (error) {
      throw new Error(`Invalid JSON for wsHeaders: ${error.message}`);
    }
  },
  implies: 'browserWsEndpoint',
}
```

### 2. Browser Connection (src/browser.ts)

Update `ensureBrowserConnected` function:

```typescript
export async function ensureBrowserConnected(options: {
  browserURL?: string;
  browserWSEndpoint?: string;
  headers?: Record<string, string>;
  devtools: boolean;
}) {
  if (browser?.connected) {
    return browser;
  }

  const connectOptions: ConnectOptions = {
    targetFilter: makeTargetFilter(options.devtools),
    defaultViewport: null,
    handleDevToolsAsPage: options.devtools,
  };

  if (options.browserWSEndpoint) {
    connectOptions.browserWSEndpoint = options.browserWSEndpoint;
    if (options.headers) {
      connectOptions.headers = options.headers;
    }
  } else if (options.browserURL) {
    connectOptions.browserURL = options.browserURL;
  }

  browser = await puppeteer.connect(connectOptions);
  return browser;
}
```

### 3. Main Entry Point (src/main.ts)

Update `getContext()` to pass new options:

```typescript
const browser = args.browserUrl || args.browserWsEndpoint
  ? await ensureBrowserConnected({
      browserURL: args.browserUrl,
      browserWSEndpoint: args.browserWsEndpoint,
      headers: args.wsHeaders,
      devtools,
    })
  : await ensureBrowserLaunched({...});
```

### 4. Documentation (README.md)

Add new options to the configuration section:

- `--browserWsEndpoint`, `-w`: WebSocket endpoint
- `--wsHeaders`: Custom headers in JSON format

Add usage examples showing:

1. Basic WebSocket connection
2. WebSocket with authentication headers
3. Comparison with browserUrl approach

## Benefits

1. **Direct WebSocket connection**: Bypass HTTP endpoint resolution
2. **Authentication support**: Pass custom headers for secured browser instances
3. **Flexibility**: Choose between HTTP URL or WebSocket endpoint based on use case
4. **API keys/tokens**: Support authenticated remote debugging scenarios

## Implementation Progress

### 1. CLI Arguments (src/cli.ts) ✅
- ✅ Added `browserWsEndpoint` argument with WebSocket protocol validation
- ✅ Added `wsHeaders` argument with JSON parsing and validation
- ✅ Configured mutual exclusivity between `browserUrl` and `browserWsEndpoint`
- ✅ Set up `implies` relationship (wsHeaders requires browserWsEndpoint)
- ✅ Updated conflicts for `executablePath` and `channel`
- ✅ Added CLI examples for WebSocket usage

### 2. Browser Connection (src/browser.ts) ✅
- ✅ Updated `ensureBrowserConnected` signature to support optional parameters
- ✅ Added logic to choose between `browserURL` and `browserWSEndpoint`
- ✅ Implemented headers passthrough when using WebSocket endpoint
- ✅ Added error handling for missing connection parameters

### 3. Main Entry Point (src/main.ts) ✅
- ✅ Updated `getContext()` to check for both `browserUrl` and `browserWsEndpoint`
- ✅ Pass all connection options to `ensureBrowserConnected`
- ✅ Maintained backward compatibility with existing code

### 4. Documentation (README.md) ✅
- ✅ Added `--browserWsEndpoint` to configuration section
- ✅ Added `--wsHeaders` to configuration section
- ✅ Created new "Connecting via WebSocket with custom headers" section
- ✅ Added example configuration with authentication header
- ✅ Documented how to get WebSocket endpoint URL

## Testing Results ✅

### Build & Code Quality
- ✅ Build succeeds: `npm run build`
- ✅ Type checking passes: `npm run typecheck`
- ✅ Code formatting passes: `npm run format`

### Manual Validation
- ✅ WebSocket URL validation works (ws:// and wss:// accepted)
- ✅ Invalid protocol rejected (http:// shows proper error)
- ✅ Mutual exclusivity enforced (browserUrl + browserWsEndpoint conflict)
- ✅ wsHeaders requires browserWsEndpoint (dependency enforced)
- ✅ Invalid JSON rejected with clear error message
- ✅ Help output displays new options correctly

### Unit Tests (tests/cli.test.ts)
- ✅ parses browserWsEndpoint with ws:// protocol
- ✅ parses browserWsEndpoint with wss:// protocol
- ✅ parses wsHeaders with valid JSON
- ✅ All 9 CLI tests passing

## Git & PR Status ✅

- ✅ Branch created: `sonkatalon/ws-endpoint-headers`
- ✅ Changes committed with conventional commit message
- ✅ Branch pushed to remote
- ✅ Ready for PR creation

**PR URL**: https://github.com/daohoangson/chrome-devtools-mcp/pull/new/sonkatalon/ws-endpoint-headers

## Summary

All planned changes have been successfully implemented and tested. The feature adds WebSocket endpoint and custom headers support while maintaining full backward compatibility with existing functionality.
