# Nostr Git Browser - Design Document

A browser-based git repository viewer and sync client using Nostr for real-time updates.

## Status: ✅ Implemented

Working prototype in `index.html` (~1000 lines, single file, no build step).

**Features working:**
- Clone repositories to IndexedDB
- Browse files with expandable tree
- View files (raw text or HTML preview)
- HTML preview with local JSON injection (fetch interception)
- Auto-sync on Nostr 30617 events from trusted pubkeys
- Multi-relay WebSocket connections with status indicators
- Add/remove repositories via modal
- Configuration persisted to localStorage
- **Git push with NIP-98 authentication** (Schnorr signatures)
- Settings modal for private key configuration
- Create commits and push to remote servers

## Overview

This project brings the functionality of [nostr-git-sync](https://github.com/JavaScriptSolidServer/nostr-git-sync) to the browser, allowing users to:

1. Subscribe to multiple git repositories via Nostr (NIP-34)
2. Clone/sync repositories to browser IndexedDB
3. Browse repository files with a visual UI
4. Auto-update when new commits are published
5. **Push commits with NIP-98 Schnorr authentication**

```
┌─────────────────┐     WebSocket      ┌─────────────────┐
│                 │◄──────────────────►│  Nostr Relays   │
│     Browser     │                    │  (30617 events) │
│                 │                    └─────────────────┘
│  IndexedDB      │     HTTP/fetch     ┌─────────────────┐
│  /sync/repo1/   │◄──────────────────►│  Git Server     │
│  /sync/repo2/   │  clone/pull/push   │  (JSS w/NIP-98) │
│                 │   (NIP-98 auth)    └─────────────────┘
└─────────────────┘
```

## Core Concepts

### NIP-34 Events

**Kind 30617** - Repository Announcement
```json
{
  "kind": 30617,
  "tags": [
    ["d", "repo-id"],
    ["name", "repo-name"],
    ["clone", "https://server.com/repo"],
    ["refs/heads/main", "abc123..."]
  ],
  "content": "Latest commit: message",
  "pubkey": "publisher-pubkey"
}
```

When a trusted pubkey publishes a 30617 event for a tracked repo, the browser syncs.

### Storage Structure

Uses IndexedDB via LightningFS (isomorphic-git's filesystem):

```
IndexedDB: nostr-git-browser
├── /sync/
│   ├── /20260202/
│   │   ├── .git/
│   │   ├── index.html
│   │   ├── gitmark.html
│   │   ├── webledgers.json
│   │   └── .well-known/txo/txo.json
│   └── /another-repo/
│       └── ...
```

### Configuration

Stored in localStorage as `nostr-git-config`:

```json
{
  "relays": [
    "wss://relay.damus.io",
    "wss://nos.lol",
    "wss://melvin.me/relay"
  ],
  "repos": {
    "20260202": {
      "cloneUrl": "https://melvin.me/public/git/sync/20260202",
      "branch": "gh-pages",
      "trusted": ["d769d2b81c051d2f2c0b437d0ffe39e00ff0f7161b520f0bd30811f4c057795f"]
    }
  }
}
```

## Architecture

### Single-File Implementation

The entire application is contained in `index.html` with inline CSS and JavaScript:

```
index.html (~750 lines)
├── <style>           # CSS (layout, components, modals)
├── <script type="module">
│   ├── Imports       # Preact, htm, isomorphic-git, LightningFS
│   ├── Config        # localStorage load/save
│   ├── Git Ops       # clone, pull, listFiles, readFile
│   ├── Nostr         # WebSocket relay connections, subscriptions
│   ├── Components    # Preact functional components
│   │   ├── Header    # App header with relay status
│   │   ├── Sidebar   # Repository list
│   │   ├── FileTree  # Expandable file browser
│   │   ├── FileView  # File content viewer with preview
│   │   ├── AddRepoModal
│   │   └── Toast     # Notifications
│   └── App           # Main component, state management
```

### Dependencies

All loaded via CDN (esm.sh) - **no build step required**:

```javascript
import { h, render } from 'https://esm.sh/preact@10.19.3';
import { useState, useEffect, useCallback, useMemo } from 'https://esm.sh/preact@10.19.3/hooks';
import htm from 'https://esm.sh/htm@3.1.1';
import git from 'https://esm.sh/isomorphic-git@1.25.3';
import http from 'https://esm.sh/isomorphic-git@1.25.3/http/web';
import LightningFS from 'https://esm.sh/@isomorphic-git/lightning-fs@4.6.0';
```

**Why Preact + htm:**
- No build tools (Webpack, Vite, etc.)
- Single HTML file
- JSX-like syntax via tagged templates
- Tiny footprint (3kb vs React's 40kb+)
- Hooks support

## Key Features

### 1. Git Operations

```javascript
// Clone repository
await git.clone({
  fs, http, dir,
  url: cloneUrl,
  ref: branch,
  singleBranch: true,
  depth: 1
});

// Pull updates
await git.pull({
  fs, http, dir,
  ref: branch,
  singleBranch: true,
  author: { name: 'browser', email: 'browser@local' }
});

// Push with NIP-98 auth
await git.push({
  fs,
  http: createAuthenticatedHttp(privkey),  // Custom HTTP client
  dir,
  url: cloneUrl,
  ref: branch
});
```

### 2. NIP-98 Push Authentication

Git push uses NIP-98 (HTTP Auth) with Schnorr signatures. Each HTTP request gets a unique signed token:

```javascript
function createAuthenticatedHttp(privkey) {
  return {
    async request({ url, method, headers, body }) {
      // Create NIP-98 event (kind 27235)
      const event = {
        kind: 27235,
        created_at: Math.floor(Date.now() / 1000),
        tags: [['u', url], ['method', method]],
        content: ''
      };
      const signed = await signEvent(event, privkey);
      const token = btoa(JSON.stringify(signed));

      return fetch(url, {
        method,
        headers: { ...headers, 'Authorization': 'Nostr ' + token },
        body
      });
    }
  };
}
```

**Key points:**
- Uses `@noble/curves` for Schnorr signatures (same as Bitcoin/Nostr)
- Each request (GET /info/refs, POST /git-receive-pack) gets its own token
- Server validates signature and checks ACL for `did:nostr:<pubkey>`
- Private key stored in localStorage (settings modal)

### 3. Nostr Relay Connections

```javascript
function connectRelays(relays, repoIds, trusted, onEvent) {
  for (const url of relays) {
    const ws = new WebSocket(url);
    ws.onopen = () => {
      // Subscribe to kind 30617 for tracked repos
      ws.send(JSON.stringify([
        "REQ", "sync",
        { kinds: [30617], "#d": repoIds, authors: trusted }
      ]));
    };
    ws.onmessage = (msg) => {
      const [type, , event] = JSON.parse(msg.data);
      if (type === 'EVENT') onEvent(event);
    };
  }
}
```

### 3. HTML Preview with Fetch Interception

When viewing HTML files, the app injects a script that intercepts `fetch()` calls for local JSON files:

```javascript
// Scan HTML for fetch() patterns
const fetchMatches = content.matchAll(/fetch\s*\(\s*['"]([^'"?]+)/g);
for (const match of fetchMatches) {
  const fetchPath = match[1];
  const json = await readFile(repoId, fetchPath);
  localFiles[fetchPath] = JSON.parse(json);
}

// Inject fetch wrapper into HTML before rendering in iframe
const fetchWrapper = '<script>' +
  'window.__localFiles = ' + JSON.stringify(localFiles) + ';' +
  'const _fetch = window.fetch;' +
  'window.fetch = (url, ...args) => {' +
  '  const name = String(url).split("?")[0];' +
  '  if (window.__localFiles[name] !== undefined) {' +
  '    return Promise.resolve(new Response(JSON.stringify(window.__localFiles[name])));' +
  '  }' +
  '  return _fetch(url, ...args);' +
  '};' +
  '<' + '/script>';
```

This allows HTML files like `index.html` (WebLedger) and `gitmark.html` to display real data from their companion JSON files stored in IndexedDB.

### 4. File Tree with Lazy Loading

```javascript
function FileTree({ files, selected, expanded, onSelect, onToggle, depth = 0 }) {
  return html`
    ${files.map(entry => html`
      <div key=${entry.path}>
        <div
          class="file-entry ${selected === entry.path ? 'selected' : ''}"
          style="padding-left: ${12 + depth * 16}px"
          onClick=${() => entry.type === 'dir' ? onToggle(entry.path) : onSelect(entry.path)}
        >
          <span>${entry.type === 'dir' ? (expanded.has(entry.path) ? '📂' : '📁') : '📄'}</span>
          <span>${entry.name}</span>
        </div>
        ${entry.type === 'dir' && expanded.has(entry.path) && entry.children && html`
          <${FileTree} files=${entry.children} ... depth=${depth + 1} />
        `}
      </div>
    `)}
  `;
}
```

## User Interface

### Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  🔗 Nostr Git Browser              ● nos.lol  ● damus  ● melvin │
├────────────────────┬────────────────────────────────────────────┤
│                    │                                            │
│  REPOSITORIES      │  index.html                         [Raw]  │
│  ───────────────   │  ─────────────────────────────────────────  │
│                    │                                            │
│  📁 20260202       │  ┌─────────────────────────────────────┐   │
│     📂 .well-known │  │         WebLedger                   │   │
│        📂 txo      │  │  2026-02-02 | Updated: 4:53 PM      │   │
│           txo.json │  │                                     │   │
│     gitmark.html   │  │  ENTRIES                            │   │
│     index.html  ◄  │  │  did:nostr:de7ecd...  1,242,035     │   │
│     mark.sh        │  │                                     │   │
│     sync.sh        │  │  TOTAL: 1,242,035 sats              │   │
│     SYSTEM.md      │  └─────────────────────────────────────┘   │
│     webledgers.json│                                            │
│                    │                                            │
│  [+ Add Repository]│                                            │
└────────────────────┴────────────────────────────────────────────┘
```

### Status Indicators

- 🟢 Connected to relay (green dot in header)
- 🟡 Syncing (spinner on repo)
- Toast notifications for sync events

### Interactions

1. **Add Repo**: Click [+ Add Repository] → Modal → Enter details → Clone starts
2. **Browse Files**: Click folder to expand, click file to view
3. **View HTML**: Toggle Raw/Preview button for rendered view
4. **Auto-sync**: Nostr events from trusted pubkeys trigger pull

## Technical Considerations

### CORS

Git servers must allow CORS for browser fetch:
- JSS has CORS enabled by default
- GitHub raw URLs work
- May need CORS proxy for some servers

### Storage Limits

IndexedDB limits vary by browser:
- Chrome: ~80% of available disk
- Firefox: ~50% of available disk
- Safari: ~1GB default

### Branch Handling

The app uses the configured branch for both clone and pull operations:
```javascript
await git.clone({ ..., ref: branch, singleBranch: true });
await git.pull({ ..., ref: branch, singleBranch: true });
```

### Security

- Only sync from trusted pubkeys (configurable per repo)
- HTML previews run in sandboxed iframe with `sandbox="allow-scripts"`
- Fetch interception only provides read access to IndexedDB files

## Running

Just open `index.html` in a browser:

```bash
cd nostr-git-browser
python -m http.server 3006
# or
npx serve .
```

Then visit `http://localhost:3006`

No build step, no npm install, no configuration files needed.

## Example: Add a Repository

1. Click **"+ Add Repository"**
2. Enter:
   - **ID**: `20260202`
   - **URL**: `https://melvin.me/public/git/sync/20260202`
   - **Branch**: `gh-pages`
   - **Trusted**: `d769d2b81c051d2f2c0b437d0ffe39e00ff0f7161b520f0bd30811f4c057795f`
3. Click **Add**
4. Repository clones to IndexedDB
5. Browse files, view index.html with live WebLedger data

## Example: Push Changes

1. Open Settings (⚙️ button)
2. Enter your **private key** (64-char hex)
3. Select a repository from dropdown
4. Click **"Create Commit & Push"**
5. Creates `browser-test.txt`, commits, and pushes to remote

The push button (↑) in repo list also works for pushing existing commits.

## Future Improvements

- [ ] Delete repository button
- [ ] File editing UI
- [ ] NIP-07 browser extension support (avoid raw privkey)
- [ ] Syntax highlighting for code files
- [ ] Commit history viewer
- [ ] Diff viewer
- [ ] Service worker for offline support
- [ ] Publish 30617 events after push
- [ ] **Bitcoin commit anchoring (gitmark)**
  - [ ] Key derivation for transaction chaining (base_key + commit_hashes)
  - [ ] Build and sign Bitcoin transactions in browser
  - [ ] Broadcast via mempool.space API
  - [ ] TXO file management (.well-known/txo/txo.json)
  - [ ] "Mark" button in UI to anchor commits to Bitcoin
- [ ] **Client-side validated smart contracts**
  - [x] Minimal first contract: "Add 500 sats" button (`contract-test.html`)
  - [ ] state.json - Contract state (balances, entries, etc.)
  - [ ] contract.js - Pure validation functions for state transitions
  - [ ] schema.json - Optional JSON Schema for structure validation
  - [ ] CLI runner to apply updates before commit
  - [ ] Trust model: pubkey signs valid state, Bitcoin anchors ordering

## References

- [isomorphic-git](https://isomorphic-git.org/) - Git implementation in JavaScript
- [LightningFS](https://github.com/isomorphic-git/lightning-fs) - IndexedDB filesystem
- [NIP-34](https://github.com/nostr-protocol/nips/blob/master/34.md) - Git over Nostr
- [NIP-98](https://github.com/nostr-protocol/nips/blob/master/98.md) - HTTP Auth with Nostr
- [nostr-git-sync](https://github.com/JavaScriptSolidServer/nostr-git-sync) - Server-side equivalent
- [@noble/curves](https://github.com/paulmillr/noble-curves) - Schnorr signatures
- [Preact](https://preactjs.com/) - Fast 3kB React alternative
- [htm](https://github.com/developit/htm) - JSX-like syntax with template literals

## License

MIT
