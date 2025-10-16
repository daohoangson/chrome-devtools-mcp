# Plan: Add WebSocket Endpoint and Headers Support

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

## Testing Considerations
- Verify WebSocket URL validation
- Test header JSON parsing (valid/invalid cases)
- Ensure headers only work with WebSocket endpoint
- Test mutual exclusivity of browserUrl and browserWsEndpoint
- Verify wsHeaders requires browserWsEndpoint
