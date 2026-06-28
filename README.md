# venezuela-ayuda-secure — Docker Sandbox Kit

A **mixin kit** for [`mawmawmaw/venezuela-ayuda`](https://github.com/mawmawmaw/venezuela-ayuda)
that hardens the development environment against npm supply-chain attacks.

## Threat model

| Attack vector | Mitigation |
|---|---|
| Malicious `postinstall` / `preinstall` hooks | `--ignore-scripts` on all `npm ci` / `npm install` runs; `.npmrc` enforces it permanently |
| Lockfile drift (dependency confusion, phantom packages) | `npm ci` with `prefer-frozen-lockfile=true` — install fails if `package-lock.json` doesn't match |
| Known CVEs in transitive deps | `npm audit --audit-level=moderate` runs at install time **and** on every sandbox start |
| Typosquatting, malware, suspicious behavior | Socket.dev CLI scans the full dependency tree at install time |
| Secret exfiltration via a compromised package | All API keys are **proxy-managed**: the container process sees placeholder env vars; the actual value is injected in-flight by the sbx proxy and never stored on disk inside the VM |
| Unrestricted outbound network access | Strict domain allowlist — only registry.npmjs.org, Supabase, OpenAI, FR-API, MapTiler, and GitHub are reachable |
| Next.js / toolchain telemetry leaking env data | `NEXT_TELEMETRY_DISABLED=1` cuts all telemetry calls; `vercel.com` is absent from the allowlist |

## Directory layout

```
venezuela-ayuda-sbx/
├── spec.yaml                  # Kit spec (mixin)
├── README.md                  # This file
└── files/
    └── workspace/
        └── .npmrc             # Hardened npm configuration (auto-copied into repo root)
```

## Prerequisites on the host

Before running `sbx run`, export the credentials that the proxy will manage:

```sh
# Required
export NEXT_PUBLIC_SUPABASE_URL="https://<project>.supabase.co"
export NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY="sb_publishable_..."
export SUPABASE_SECRET_KEY="sb_secret_..."           # or SUPABASE_SERVICE_ROLE_KEY

# Optional – AI classification
export OPENAI_API_KEY="sk-..."

# Optional – face-recognition dedup
export FR_API_KEY="..."

# Optional – migration scripts / backup
export SUPABASE_DB_URL="postgresql://..."

# Optional – MapTiler vector tiles
export NEXT_PUBLIC_MAP_STYLE_URL="https://api.maptiler.com/maps/streets/style.json?key=..."
```

`NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` are
**not** secrets (they're safe for the browser), so they are set directly as
environment variables rather than going through the credential proxy.

The secret vars (`SUPABASE_SECRET_KEY`, `OPENAI_API_KEY`, `FR_API_KEY`,
`SUPABASE_DB_URL`) are proxy-managed: their values live only on the host and
are substituted in-flight on outbound requests. The container never sees the
raw bytes.

## Usage

```sh
# Create a new sandbox from the venezuela-ayuda repo with this kit applied
sbx run claude \
  --kit ./venezuela-ayuda-sbx/ \
  git+https://github.com/mawmawmaw/venezuela-ayuda.git

# Or, if you already have a local clone:
sbx run claude \
  --kit ./venezuela-ayuda-sbx/ \
  /path/to/venezuela-ayuda

# Apply to an existing (already-created) sandbox without recreating it:
sbx kit add <sandbox-name> ./venezuela-ayuda-sbx/
```

## What happens at install time

1. System packages (`ca-certificates`, `curl`, `gnupg`) updated.
2. Node.js 22 LTS installed from the signed NodeSource repository.
3. Socket.dev CLI installed globally (`--ignore-scripts`).
4. `npm ci --ignore-scripts --audit=false` runs in the workspace (lockfile-frozen).
5. `npm audit --audit-level=moderate` runs; **build fails if moderate+ CVEs exist**.
6. `socket scan create .` runs a behavioral scan of the dependency tree.
7. `.env.local` is written from the proxy-injected values (only if not already present).

## What happens on every sandbox start

- `npm audit` re-runs in the background against the installed tree, logging to
  `/tmp/npm-audit.log`. This catches advisories published after the initial install.

## Adding new npm dependencies

```sh
# Always pass --ignore-scripts. Never omit it.
npm install --ignore-scripts <package>

# Verify the addition
npm audit
socket scan create .

# Review the lockfile diff before committing
git diff package-lock.json
```

## Debugging

```sh
# See all outbound requests and whether they were allowed or blocked
sbx policy log

# Inspect the installed state inside the sandbox
sbx exec <sandbox-name> -- which socket
sbx exec <sandbox-name> -- cat /tmp/npm-audit.log
sbx exec <sandbox-name> -- cat /home/agent/workspace/.npmrc
```

## Kit validation

```sh
sbx kit validate ./venezuela-ayuda-sbx/
sbx kit inspect ./venezuela-ayuda-sbx/
```
