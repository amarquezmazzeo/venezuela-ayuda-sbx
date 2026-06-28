# venezuela-ayuda-shell — Docker Sandbox Kit (`sbx run shell`)

A **mixin kit** for [`mawmawmaw/venezuela-ayuda`](https://github.com/mawmawmaw/venezuela-ayuda)
targeting `sbx run shell` — the bare Bash login shell agent.

## Differences from the `claude` agent variant

| | `claude` kit | `shell` kit |
|---|---|---|
| Agent | Claude Code (pre-installed) | Bash login shell (no agent) |
| `memory:` block | ✅ CLAUDE.md injected | ❌ not applicable |
| Startup audit | Background (agent reads log) | Background → `/tmp/npm-audit.log`; MOTD shows warning |
| MOTD | None | Printed on every shell login |
| `npm run dev` | Agent can start it | You run it manually |
| Node.js | Pre-installed in agent image | Installed by kit (NodeSource, signed) |

Everything else — credential proxy, domain allowlist, `--ignore-scripts`, frozen lockfile, Socket.dev scan — is identical.

## Threat model

| Attack vector | Mitigation |
|---|---|
| Malicious `postinstall` / `preinstall` hooks | `--ignore-scripts` on all `npm ci` / `npm install` runs; `.npmrc` enforces it permanently |
| Lockfile drift (dependency confusion) | `npm ci` with `prefer-frozen-lockfile=true` — fails if `package-lock.json` doesn't match |
| Known CVEs in transitive deps | `npm audit --audit-level=moderate` at install + every sandbox start (MOTD warns if issues) |
| Typosquatting / malware behavior | Socket.dev CLI behavioral scan at install time |
| Secret exfiltration via compromised package | All API keys are **proxy-managed** — value never touches the VM's memory or disk |
| Unrestricted outbound network | Strict domain allowlist; `vercel.com` / Next.js telemetry disabled |

## Directory layout

```
venezuela-ayuda-sbx-shell/
├── spec.yaml                  # Kit spec (mixin, shell-targeted)
├── README.md                  # This file
└── files/
    └── workspace/
        └── .npmrc             # Hardened npm config (auto-copied to repo root)
```

## Prerequisites on the host

```sh
# Required
export NEXT_PUBLIC_SUPABASE_URL="https://<project>.supabase.co"
export NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY="sb_publishable_..."
export SUPABASE_SECRET_KEY="sb_secret_..."

# Optional
export OPENAI_API_KEY="sk-..."
export FR_API_KEY="..."
export SUPABASE_DB_URL="postgresql://..."
export NEXT_PUBLIC_MAP_STYLE_URL="https://api.maptiler.com/maps/streets/style.json?key=..."
```

Secret vars (`SUPABASE_SECRET_KEY`, `OPENAI_API_KEY`, `FR_API_KEY`, `SUPABASE_DB_URL`)
are proxy-managed: the sandbox process sees placeholder env var names; actual values
are substituted in-flight on outbound HTTPS requests only.

## Usage

```sh
# Create a new shell sandbox from the venezuela-ayuda repo with this kit
sbx run shell \
  --kit ./venezuela-ayuda-sbx-shell/ \
  git+https://github.com/mawmawmaw/venezuela-ayuda.git

# Or from a local clone
sbx run shell \
  --kit ./venezuela-ayuda-sbx-shell/ \
  /path/to/venezuela-ayuda

# Apply to an already-created sandbox without recreating it
sbx kit add <sandbox-name> ./venezuela-ayuda-sbx-shell/
```

Once inside the shell you'll see the MOTD, then you're ready:

```sh
npm run dev          # dev server at http://localhost:3000
npm run build        # production build
npm run lint         # ESLint
npm test             # node --test scripts/*.test.mjs

# Migration scripts
node scripts/apply/check-migrations.mjs
node scripts/backup-db.mjs
```

## What happens at install time

1. System packages (`ca-certificates`, `curl`, `gnupg`, `git`) updated.
2. Node.js 22 LTS installed from the **signed** NodeSource repository.
3. Socket.dev CLI installed globally (`--ignore-scripts`).
4. `npm ci --ignore-scripts --audit=false` — lockfile-frozen, no hooks.
5. `npm audit --audit-level=moderate` — **fails sandbox creation on moderate+ CVEs**.
6. `socket scan create .` — behavioral scan, non-blocking but printed.
7. `.env.local` written from proxy-injected values (`0600`, `onlyIfMissing`).
8. Bash profile and MOTD script dropped into `/home/agent/`.

## What happens on every sandbox start

`npm audit` re-runs in the background and writes to `/tmp/npm-audit.log`.
The MOTD (printed when you log in) shows a warning line if issues were found.

## Adding new dependencies

```sh
# Always --ignore-scripts. Never omit it.
npm install --ignore-scripts <package>

# Verify
npm audit
socket scan create .

# Review lockfile diff before committing
git diff package-lock.json
```

## Debugging

```sh
# See all outbound requests and allow/block decisions
sbx policy log

# Inspect inside a running sandbox
sbx exec <sandbox-name> -- cat /tmp/npm-audit.log
sbx exec <sandbox-name> -- cat /home/agent/workspace/.npmrc
sbx exec <sandbox-name> -- node --version
sbx exec <sandbox-name> -- which socket
```

## Kit validation

```sh
sbx kit validate ./venezuela-ayuda-sbx-shell/
sbx kit inspect ./venezuela-ayuda-sbx-shell/
```
