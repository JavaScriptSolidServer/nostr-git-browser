# INIT.md - LLM Onboarding

Quick context for starting a new session on this project.

## What This Is

Browser-based git client using Nostr for sync events and Bitcoin for anchoring. No build step - single HTML files with CDN imports.

## Key Files

| File | Purpose |
|------|---------|
| `index.html` | Main app - clone repos, browse files, push with NIP-98 |
| `contract-test.html` | Minimal contract experiment - "Add 500 sats" button |
| `DESIGN.md` | Full architecture, features, future work |

## Tech Stack

- **Preact + htm**: React-like UI, no build step
- **isomorphic-git**: Git in browser
- **LightningFS**: IndexedDB filesystem
- **@noble/curves**: Schnorr signatures (NIP-98)
- All via `esm.sh` CDN imports

## What Works

1. Clone git repos to IndexedDB
2. Browse files, preview HTML with local JSON injection
3. Git push with NIP-98 authentication (Schnorr signatures)
4. Auto-sync on Nostr 30617 events
5. Minimal contract: add sats to webledger, commit, push

## Key Patterns

**NIP-98 Auth** (each HTTP request signed):
```javascript
const event = { kind: 27235, tags: [['u', url], ['method', method]] };
const signed = await signEvent(event, privkey);
headers['Authorization'] = 'Nostr ' + btoa(JSON.stringify(signed));
```

**Contract pattern** (client-side validation):
```javascript
function addSats(state, pubkey, amount) {
  // Modify state
  state.updated = Math.floor(Date.now() / 1000);
  return state;
}
// Write to IndexedDB → git commit → push
```

## Testing

```bash
cd /home/melvin/projects/nostr-git-browser
python -m http.server 3006
# Open http://localhost:3006
# Or http://localhost:3006/contract-test.html
```

## Related Repos

- `/home/melvin/wl/20260202` - WebLedger data repo (synced via Nostr)
- `nostr-git-sync` - Server-side daemon that pulls on 30617 events

## Next Steps (from DESIGN.md)

Priority candidates:
- [ ] NIP-07 browser extension support (avoid raw privkey)
- [ ] Publish 30617 events after push
- [ ] Bitcoin commit anchoring (gitmark)
- [ ] More contract actions (transfer, custom amounts)

## Config

Repos stored in `localStorage` as `nostr-git-config`. Private key in `nostr-git-privkey`.
