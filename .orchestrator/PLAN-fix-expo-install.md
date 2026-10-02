# PLAN - fix-expo-install
Task: Fix failed `create-expo-app` dependency install (npm ETIMEDOUT, no node_modules)

## Diagnosis (Phase 0)
- `npm install` fetched OK but extremely slowly (80-240s/tarball) then read ETIMEDOUT
- Registry reachable: ~687 kB/s (slow but usable) — transient timeouts, not proxy (proxy=null)
- npm 12.2.0 via mise + Node 26.10.0; npm fetch-retries=2, timeout 300s — too fragile for this link
- No node_modules, no lockfile; package.json = fresh SDK 57 template; .vscode fix intact
- Bun 1.4.2 available — parallel fetcher, far more resilient on slow links

## Subtasks
1. [orchestrator] `bun install` (creates bun.lock) — tdd:false
   - Verify: node_modules/expo exists, `bunx expo-doctor` clean-ish
2. [verification] `npx tsc --noEmit` + `npx expo lint` (or bunx equivalents)
   - Evidence: paste output summaries

## Edit order
Install → doctor --fix if needed → typecheck/lint
