# NM Compute integration research

Observed on NM Workstation (`ip-172-31-41-57`) on **2026-10-06, about 21:45–21:51 UTC**. This note separates what the machine and live hosts actually returned from inferences and from the October 2026 product note.

NM Compute **is on this host**, as source code, the staff fleet origin, and the capacity collector. It is **not** a GPU scheduler running on this box. The customer site and API run on Cloudflare. This machine has no NVIDIA GPU, no Docker, no Kubernetes, no Ray, no Slurm, and no SkyPilot.

## How to read this

| Label | Meaning |
| --- | --- |
| **Fact** | Seen in a file, process, cron, DNS record, or HTTP response during this pass. |
| **Hypothesis** | A reading that fits those facts but was not proven (no authenticated production call, no Cloudflare dashboard). |
| **Team context** | From the task brief, not from this disk. The Ornn quote is team context. Nothing named Ornn exists under `/home/ubuntu/nm-compute` or `/home/ubuntu/gpu-capacity-board`. |

Secrets are omitted: Cloudflare tokens, D1 ids, origin bearer tokens, provider keys, SSH endpoints, and cloud tenant or project ids. Those live in mode-600 files and in `nm-compute/docs/OPERATIONS.md` on the workstation. Do not copy them into a client.

## Host inventory

**Fact.** Hostname `ip-172-31-41-57` (`ip-172-31-41-57.ec2.internal`). Ubuntu 22.04.5 LTS, kernel `6.8.0-1063-aws`, EC2 type `c7i.8xlarge` (32 vCPU, 61 GiB RAM). Uptime about 10 days at the time of the check. Root filesystem 605G of 679G used (90%). `NM_HEARTBEAT.md` (cron `*/5`) reports the same disk warning and `GPU: 0/0 busy (no-gpu)`. `nvidia-smi` is not installed. `lspci` showed no NVIDIA device.

**Fact.** Not installed, or not on `PATH`: Docker, Podman, `kubectl`, Helm, k3s, Ray, Slurm (`sinfo`), SkyPilot. `ss` showed no Redis, RabbitMQ, or scheduler port. The only long-lived listeners that matter for this product are SSH (`22`), nginx (`80` and `443`), and a loopback Python static server on `127.0.0.1:8765` (`python3 -m http.server` for `/tmp/nm-ui-preview`). Nothing was listening on `8787`, `4174`, `8002`, or `8003`.

**Fact.** systemd (system): `nginx.service` active; `nm-memwatch.service` active (memory snapshot of `user.slice` every 60s, `/usr/local/sbin/nm-memwatch`). User systemd: `genet-feishu-eval-sync.service` (GenET evaluation workbook sync, not Compute). No `nm-compute` unit.

**Fact.** User crontab lines that are actually scheduled (others are commented as parked or paused):

- `*/5` `/home/ubuntu/nm-heartbeat.sh`
- `*/10` `/home/ubuntu/.nm-monitor/nm-deadman.sh`
- hourly `genet-dev/oauth_warn.sh`, half-hourly `genet-dev/credit_warn.sh`
- `*/2` `/home/ubuntu/gpu-capacity-board/poll.sh` (purchasable capacity)
- `* * * * *` `/home/ubuntu/gpu-capacity-board/fleet.sh` (running fleet)

**Fact.** Node used by local tools is `~/.nvm/versions/node/v22.22.0`. `nmc` 0.2.1 is installed at `/home/ubuntu/.local/bin/nmc`. There is no `~/.config/nmc` credentials directory. `/usr/bin/nm` is GNU binutils, which is why the CLI is named `nmc`.

**Fact.** nginx vhosts in `/etc/nginx/sites-enabled`:

| Server name | What it does on this box |
| --- | --- |
| `gpu-monitor-origin.neural-motion.org` | This host. DNS A record matches this instance. Port 80 only serves ACME. Port 443 includes `/etc/nginx/snippets/gpu-fleet-snapshot.conf` (mode 600, not readable here) and returns 404 for every other path. |
| `platform.neural-motion.org` and `.com` | Config proxies to `127.0.0.1:8002`, which is **not listening**. Public DNS for `platform.neural-motion.org` is a CloudFront-style set of addresses, not this instance. |
| `dev.platform.neural-motion.org` | Config proxies to `127.0.0.1:8003`, also not listening. |
| `neural-motion.com` / `.org` and `www` | Static site config on this nginx. Public DNS for `neural-motion.org` is elsewhere (CloudFront-style). |
| default | `server_name _` |

An unauthenticated `GET https://gpu-monitor-origin.neural-motion.org/_nm/fleet-snapshot.json` returned nginx **404**. The public board at `https://gpu-capacity-board.pages.dev/data/fleet.json` returned **200** with the same summary as the local snapshot. The origin token was not used.

## Where the code is

**Fact.** Product tree: `/home/ubuntu/nm-compute`, git branch `compute-router`, HEAD `3b4ebe8` (“English and Simplified Chinese across the web app”). **No `origin` remote is configured.** A second tree, `/home/ubuntu/nm-compute-deploy`, is a detached checkout of the same commit.

**Fact.** Capacity collector: `/home/ubuntu/gpu-capacity-board`, remote `https://github.com/NMotion-Team/gpu-capacity-board.git`. Cron commits capacity snapshots and deploys the Pages project `gpu-capacity-board`.

**Fact.** Layout of `/home/ubuntu/nm-compute` (version `0.1.0` in `package.json`):

| Path | Role |
| --- | --- |
| `apps/web` | Vite, React 18, TypeScript site |
| `worker` | Edge (`worker/src/edge.js`) and core (`worker/src/core.js`), D1 migrations `0001`–`0005` |
| `cli` | Python package `nmc` 0.2.1 (`cli/src/nmc_cli/__init__.py`) |
| `docs` | `API.md`, `ARCHITECTURE.md`, `OPERATIONS.md`, `LAUNCH-CHECKLIST.md` |
| `wrangler.toml` | Pages project `nm-compute` |
| `deploy/compute-preview/wrangler.toml` | Pages project `nm-compute-preview` |
| `worker/wrangler.core.toml` | Core Worker, cron `* * * * *`, no public hostname |
| `scripts/deploy.sh`, `set-secrets.sh`, `wrangler.sh` | Deploy from this workstation using `~/.cloudflare-nmc.env` (mode 600) |

Local dev (not running during this pass) is `npm run dev`: API on `127.0.0.1:8787` with `NM_COMPUTE_MOCK_PROVIDER=1` and `DEV_AUTH=1`, Vite on `127.0.0.1:4174`. The README says to unset `HYPERBOLIC_API_KEY` so the workstation shell key does not enter the dev server.

## As-is architecture

**Fact.** From `README.md` and `docs/ARCHITECTURE.md`, the intended shape is:

```
browser or nmc
        │  https://compute.neural-motion.org
        ▼
Cloudflare Pages "nm-compute"     (SPA + edge worker, no secrets)
        │  service binding CORE, /api/* forwarded unchanged
        ▼
Worker "nm-compute-core"          (no public hostname)
        ├─ D1 "nm-compute"
        ├─ cron every minute: lifecycle, metering, offer refresh, reconciliation
        ├─ Clerk JWT check (issuer https://clerk.neural-motion.org)
        ├─ Stripe webhook and credit ledger
        └─ provider adapters
Preview is a second pair: Pages "nm-compute-preview" → Worker "nm-compute-core-preview" → D1 "nm-compute-preview"
https://compute-preview.neural-motion.org
```

DNS for both Compute hostnames is Cloudflare (`2606:4700:310c::…`), which matches the Route53 CNAME-to-Pages design in `wrangler.toml`. `neural-motion.org` is not a Cloudflare zone, so the public names stay on Pages.

**Fact.** Adapters registered in `worker/src/providers/all.js`: RunPod, Prime Intellect, Vast.ai, SF Compute, Nebius, Hyperbolic. Contract in `worker/src/providers/registry.js`: `kind` is `on_demand` or `reserved`; each adapter can list, quote, create, poll, and terminate. Customer instances must be named so the adapter only touches NM Compute machines. There is no queue. A launch is a synchronous `POST /api/instances`. The cron is the only scheduler, and it meters and reconciles instances the API already created.

**Fact.** A second data plane lives on this workstation and is only partly wired in:

```
cron fleet.sh (every minute)  ──SSH read-only + cloud list──►  data/fleet.json
        │
        ▼
fleet_publish.py → /var/lib/gpu-capacity-board/fleet.json
        │
        ▼
nginx on this host (Bearer, exact path)  ←── Pages worker of gpu-capacity-board
        │
        └─ also: core Worker secret FLEET_ORIGIN_*  →  GET /api/fleet  (staff)
```

`worker/src/fleet.js` proxies that origin for staff. It requires `updated_at`, `hosts`, and `tasks`, caps the body at 500000 bytes, and returns the origin JSON unchanged. `nmc fleet` calls `GET /api/fleet`.

Purchasable stock is a **different** snapshot. `poll.sh` writes `data/latest.json` and `data/canonical.json` (schema: `provider`, `region`, `platform`, `preset`, `gpu_count`, `available`, `level`, `note`). That catalogue is rendered into `index.html` and pushed to `https://gpu-capacity-board.pages.dev/`. **It is not an NM Compute API.**

**Fact.** Product-family entry points in `apps/web/src/components/brand/family.ts`:

| Entry | Status in this tree |
| --- | --- |
| Compute | This site (`/`) |
| Data | External link `https://platform.neural-motion.org` (`apps/web/src/config/company.ts`). Public DNS does not point at this workstation. `curl` returned HTTP 200. |
| Eval | Planned. Shown in the header, no URL. |
| Autoresearch | Planned. Shown in the header, no URL. |
| Docs | Compute docs (quickstart, CLI, API, billing, SSH) |

Home copy says compute is available, data is on the data platform, and evaluation and autoresearch are next.

### What is actually deployed

`docs/OPERATIONS.md` is dated **2026-10-05** and says: production core is an earlier build; production Pages still serves the old build; preview’s domain exists but nothing is deployed there; migrations `0004` and `0005` are not applied; there are no Stripe keys and no production provider keys; provider modes default to Vast `off` and the other five `listed`; no adapter has ever created a machine. That file is now stale relative to the live preview host. Treat the bullets below as the newer observation.

**Fact. Preview, 2026-10-06 21:48 UTC.**

- `GET https://compute-preview.neural-motion.org/api/health` → **200** `{"ok":true,"database":true}`
- `GET /api/public/status` → **200**, `access.mode = "restricted"`, `settingsSource = "live"`, components `api ok`, `database ok`, `payments disabled`, `launches ok`. The only provider row was Hyperbolic, `mode=listed`, `launchable=false`, `status=ok`. `markupBps` 1000.
- `GET /api/public/offers` and `/api/public/gpus` → **401** `sign_in_required` (matches restricted mode in current `worker/src/api/router.js`).
- `GET /downloads/nmc-0.2.1-py3-none-any.whl` → **200**, `application/octet-stream`, 27729 bytes. SHA-256 `ecd6b380217244a067b85abf920e5fff07e2a779124b0204265d3de1e9ee5364`, identical to `/home/ubuntu/nm-compute/apps/web/public/downloads/nmc-0.2.1-py3-none-any.whl`.

**Fact. Production, same window.**

- `GET https://compute.neural-motion.org/` → **200** HTML, title `NM Compute · Neural Motion` (466-byte shell plus hashed assets).
- `GET /api/health` and `GET /api/public/status` → **401** `{"error":"unauthorized"}`. Response headers match `worker/src/http.js` (`cache-control: private, no-store`, `vary: Authorization`, `cdn-cache-control: no-store`, `x-content-type-options: nosniff`). No `cf-access` header.
- `GET https://nm-compute.pages.dev/api/health` → **404** `not_found`, which is what current `edge.js` does for the production Pages hostname (API is not served on `*.pages.dev`).
- `GET /downloads/nmc-0.2.1-py3-none-any.whl` on the production hostname → **200** `text/html`, the same 466-byte SPA shell, not the wheel.

**Hypothesis.** Preview is running this tree (public health, restricted offers, wheel 0.2.1). Production’s API Worker is still the earlier build described in the runbook: current source answers `/api/health` with no credentials (`auth: "none"`), and the live production host does not. The HTML shell on production can be an older Pages deploy. This was not confirmed in the Cloudflare dashboard, and no signed-in production request was sent (no `nmc` credentials on this account).

**Fact.** Staff fleet snapshot at 21:48–21:49 UTC, both `data/fleet.json` and `https://gpu-capacity-board.pages.dev/data/fleet.json` (schema 1):

| | |
| --- | --- |
| Running GPUs | 34 |
| Busy / idle / unknown | 16 / 18 / 0 |
| Unreachable hosts | 0 |
| Stopped GPUs (not in the running count) | 16 |
| Hosts / tasks in the snapshot | 6 online hosts, 1 running task |

Hosts in that file: 2× GCP A100 (us-central1, 1 GPU each, on-demand) and 4× Nebius H100 (eu-north1, 8 GPUs each, on-demand). The task is a training job named `GenET · full` on 16 GPUs. Cloud inventory login for the fleet list was failing: GCP last success `2026-10-05T22:16:03Z`, Nebius last success `2026-10-05T09:52:02Z`, both “云账号登录失效”. RunPod’s cloud list was ok at `2026-10-06T21:46:03Z`. Notices say GCP and Nebius machine rows were kept and that SSH sampling was still live. No RunPod host was in this snapshot.

**Fact.** Purchasable catalogue `data/latest.json` / `data/canonical.json` at `2026-10-06T21:48:15Z`, 231 rows. Providers marked ok on that poll: AWS, Hyperbolic (`hb`), Prime Intellect, RunPod. Nebius and GCP were **not** ok (same class of auth failure). The board keeps the previous Nebius and GCP numbers and labels the source stale; those numbers are from the last success (`2026-10-05T09:54:17Z` Nebius, `2026-10-05T22:18:15Z` GCP), not a fresh read. Prime Intellect’s status was ok and **zero** canonical rows use `provider=prime`. The board README says Prime’s `gpuCount` is the shape size, not remaining stock, so it is not counted as released capacity.

**Fact.** Nebius `available` in `poll.py` is cards, computed as nodes × GPUs per preset. Among B300 rows in the retained snapshot: most Nebius `gpu-b300-sxm` rows had `available` 0; one eu-west2 1-GPU shape had 8 cards at `AVAILABILITY_LEVEL_HIGH`; one us-north1 1-GPU shape had 3 cards at `LOW`. RunPod B300 rows (EU-NL-1, EUR-IS-1, US-WA-2, plus a US-WA-2 MIG SKU) have **no** card count; the note says RunPod only returns a stock tier. There is no Amsterdam 56× B300 row.

## How a client integrates today

There is no job queue. A client uses HTTPS JSON, or `nmc`, which is a thin client of the same API.

**Fact.** Base URL in the docs and CLI default: `https://compute.neural-motion.org/api`. Auth is `Authorization: Bearer`. Cookies are not used.

| Caller | Credential | What it can do in this tree |
| --- | --- | --- |
| Website | Clerk session JWT (`azp` must be the site origin) | All scopes. Creating API keys, checkout, billing portal, and `/api/admin/*` are session-only. |
| `nmc` | API key `nmc_` + 43 chars. SHA-256 stored in D1. File `~/.config/nmc/credentials.json` mode 0600. Env `NMC_TOKEN`, `NMC_API_URL`. | `compute:read`, `compute:write`, `billing:read` as chosen when the key is created. |
| Anonymous | None | Health, public status, Stripe webhook. Offers and GPU summaries only while `access_mode` is `open`. |

**Fact.** While `access_mode` is `restricted` (the default, and what preview returned), offers need a staff account, a verified email at an allowed domain (default `neural-motion.com`), or an allowlisted verified email. Everyone else gets `403 access_restricted`. `nmc logout` still works via `POST /api/api-keys/revoke-current`.

**Fact.** CLI commands in `cli/src/nmc_cli/main.py`: `login`, `logout`, `whoami`, `gpus`, `launch`, `instances`, `instance`, `ssh`, `terminate`, `balance`, `ledger`, `topup`, `ssh-keys`, `fleet`. `nmc gpus` is documented to work without login; that is only true when access is open. On today’s preview it is not. `nmc fleet` is staff-only. `nmc topup` prints the billing page and does not charge. Install documented as:

```bash
pipx install https://compute.neural-motion.org/downloads/nmc-0.2.1-py3-none-any.whl
```

That production URL currently returns HTML. The same wheel **is** served by the preview host, and its hash matches the tree. A client pointed at preview would set `NMC_API_URL=https://compute-preview.neural-motion.org` (or `nmc login` saves the URL). Preview keys and production keys are different D1 databases and must not be mixed; session `azp` is checked the same way.

**Fact.** Routes a client would actually call (from `worker/src/api/router.js`):

| Method and path | Access |
| --- | --- |
| `GET /api/health` | Public in this tree |
| `GET /api/public/status` | Public. No prices. |
| `GET /api/public/offers`, `GET /api/public/gpus` | Public only if access is open; otherwise `compute:read` |
| `GET /api/me`, `GET /api/offers`, `GET /api/instances`, `GET /api/instances/:id` | Signed in; reads need `compute:read` |
| `POST /api/instances`, `POST /api/instances/:id/terminate` | `compute:write` |
| `GET/POST /api/ssh-keys`, `DELETE /api/ssh-keys/:id` | read / write |
| `GET /api/billing`, `/billing/ledger`, `/billing/usage` | `billing:read` |
| `POST /api/billing/checkout`, `/billing/portal` | Website session |
| `GET /api/fleet` | Staff and `compute:read` |
| `POST /api/stripe/webhook` | Stripe signature, not a client |

Money in the API is integer micro-dollars. Customer price is provider cost plus markup (`markup_bps` 1000 → +10%) rounded up to the cent. Reserved blocks (SF Compute in this code) are prepaid for whole hours and are not metered per second. On-demand usage is posted by the cron about every 5 minutes.

**Fact.** The running-fleet document a staff client receives is the capacity board’s `fleet.json`, not a Compute-specific schema. A capacity client that wants “what can we buy” cannot get it from `/api/fleet` or `/api/public/offers`. Those are different datasets.

## Gap versus the October 2026 strategy

**Team context.** Near-term NM Compute is an authenticated website and CLI, real fleet and capacity data, and entry points for Data, Eval, and Autoresearch. Billing, ordering, provisioning, SSH, and resale are not required for v1. Compute is the acquisition wedge. Primitives should be model-agnostic and usable by agents.

**Fact.** The tree is already a resale router: credits, Stripe, launch, SSH, per-second metering, and a 10% markup. The website’s own docs pages are quickstart, CLI, API, billing, and SSH. That is ahead of the v1 the strategy describes, and production is behind the tree.

**Fact.** What v1 asks for, mapped onto what exists:

| Strategy need | On the machine today |
| --- | --- |
| Authenticated website | Clerk is specified. Preview enforces restricted access. Production API returns 401 even for health. |
| CLI | `nmc` 0.2.1 is real and installable from preview. Default URL is production, where the wheel and the public API do not match this tree. |
| Real fleet data | Live SSH snapshot, 34 running GPUs, published on the public board and proxied to staff `GET /api/fleet`. GCP and Nebius cloud logins on this box are expired, so inventory metadata is stale even though SSH samples are fresh. |
| Real capacity data | Collected here every 2 minutes, published as a separate public site. Not in the Compute API. Nebius and GCP capacity numbers are retained from 5 Oct. |
| Data entry point | Link to `platform.neural-motion.org`. |
| Eval and Autoresearch entry points | Names in the header only. |
| No billing / order / SSH / resale for v1 | Implemented anyway, and provider modes are designed so `listed` shows offers without allowing launch. Preview currently lists only Hyperbolic, not launchable, and payments are disabled. |

## MVP

Ship a read API and a CLI on the preview host first, then replace production Pages and the production core together. Do not turn on launch, Stripe, or SSH for this cut. The code for those can stay; `access_mode=restricted`, `launches` left unused by the client, and provider modes `listed` or `off` already keep them from placing machines.

### 1. Make one authenticated read surface

Keep Clerk (website) and `nmc_` keys (agents and the CLI). Scopes for v1: `compute:read` is enough. Keep issuing keys on the website (`POST /api/api-keys` is session-only). Do not require `compute:write` or `billing:read` for the default key.

Add one read model that the website and `nmc` share. Suggested `GET /api/capacity` (name is a recommendation), `compute:read` while access is restricted:

```json
{
  "generatedAt": "2026-10-06T21:48:15Z",
  "fleet": { "runningGpus": 34, "busyGpus": 16, "idleGpus": 18, "updatedAt": "…" },
  "sources": [
    { "provider": "nebius", "ok": false, "lastSuccessAt": "2026-10-05T09:54:17Z" }
  ],
  "offers": [
    {
      "id": "ornn:b300:amsterdam:56:3mo",
      "provider": "ornn",
      "gpuFamily": "B300",
      "gpuCount": 56,
      "region": "amsterdam",
      "country": "NL",
      "billing": "reserved",
      "term": "3mo",
      "pricePerGpuHourMicros": 7500000,
      "launchable": false,
      "source": "manual_quote",
      "asOf": "2026-10-06"
    }
  ]
}
```

Populate it from three inputs that already exist or are a quote, not from the launch adapters:

1. **Fleet summary** from the origin `fleet.json` this host already publishes. Staff can keep the full document on `GET /api/fleet`. The customer and agent view should be the summary (counts, GPU model, region, busy/idle), not hostnames, SSH targets, or task owners.
2. **Market stock** from `gpu-capacity-board` `canonical.json`. Carry `lastSuccessAt` per source so a caller can see that Nebius and GCP are retained snapshots. Re-login on this box (`nebius profile update`, and the GCP login the board uses) before calling those numbers current.
3. **Manual reserved quotes**, starting with Ornn. A row in D1 or a JSON document the core reads. `launchable: false`. No `POST /api/instances`.

`/api/public/offers` should stay the launch catalogue. Mixing “we can place this on RunPod” with “a vendor quoted us a quarter of a cluster” will make an agent call `nmc launch` on something that cannot launch.

### 2. CLI

Point the v1 install at the host that actually serves the wheel until production is updated:

```bash
pipx install https://compute-preview.neural-motion.org/downloads/nmc-0.2.1-py3-none-any.whl
nmc login          # key from the preview site; URL is saved with the key
nmc capacity       # new; --json for agents
nmc fleet          # staff, full running snapshot, already implemented
```

Leave `launch`, `ssh`, `terminate`, `balance`, and `ledger` in the binary. Document them as not part of v1, and do not grant `compute:write` on the default key. `nmc gpus` stays the launch catalogue so it does not silently mean “reserved quotes”.

Agents should prefer `--json`. The API already returns stable `error` codes (`sign_in_required`, `access_restricted`, `insufficient_scope`, `unauthorized`).

### 3. Website

Authenticated pages: fleet summary, capacity table (with stale badges), and the family nav that already exists. Data stays a link to `https://platform.neural-motion.org`. Eval and Autoresearch stay visible and unlinked until those products have URLs. Do not build checkout.

### 4. Promote the build that preview is already running

**Hypothesis, operational.** Production still serves the old API and an HTML shell at the wheel URL. The next deploy from this workstation (`scripts/deploy.sh`) applies migrations `0004_instance_health.sql` and `0005_access_allowlist.sql` and then updates core and Pages. Do that for preview first if anything here is still unreleased, check `/api/health`, `/api/public/status`, and the wheel hash, then production. Until that deploy, clients should use the preview origin explicitly. The workstation checkout has **no git remote**; publish `compute-router` to a remote before that deploy so production is not “whatever was on this disk”.

Also fix the two expired cloud logins on this host before advertising fleet or Nebius capacity as live. Disk at 90% is from the heartbeat file; the capacity logs (`poll.log` about 6 MB, `fleet.log` about 4.6 MB in the repo directory) are not the bulk of it, but the collector should keep running.

### 5. Do not build in this cut

Ordering, card charges, SSH, instance lifecycle, and resale markup. Provider keys stay off the production core, which the runbook already says is the case. `launches: ok` on preview status means the settings flag, not that a machine can be created; Hyperbolic was `launchable: false`.

## Onboarding a reserved cluster (Ornn-like)

**Team context.** Example quote: Ornn, 56× B300, Amsterdam, $7.50 per GPU-hour for 3 months, $8.50 per GPU-hour as a 1-month option. Not present in the capacity snapshot or the Compute tree.

**Fact.** The only reserved provider in code is SF Compute (`worker/src/providers/sfcompute.js`, migration `0003_reserved_instances.sql`). That model is a **customer-prepaid block**: the client pays `price × hours` up front, SF fills an order, a VM is attached, and early termination does not refund the block. Metering refuses reserved instances. That is a marketplace purchase, not “Neural Motion holds a cluster and routes work onto it.”

**Recommendation.** Do not implement Ornn as another SF Compute adapter in v1. Add a held-capacity record the read API can return.

1. **v1, manual quote.** One or two non-launchable offers, as in the JSON above (`3mo` at 7_500_000 micro-dollars per GPU-hour, `1mo` at 8_500_000). Source `manual_quote`, `asOf` the quote date, `availableCount` 56, region Amsterdam, country `NL`. Show it beside live stock, labeled as a quote, not as spare GPUs the poll just found. An agent can read it and a human can order it outside the product.
2. **When Ornn has a list API.** Add an adapter whose `kind` is `reserved`, `canList` is true, and `createInstance` always fails. Register it in `all.js` and set its provider mode to `listed`. The existing cron offer refresh can then update stock without a launch path. Map their identifier, region, GPU count, term, and price into the capacity offer, not into `POST /api/instances`.
3. **When ordering is in scope (after v1).** Model NM-held reservations separately from customer instances: a pool (56× B300, term, price, start, end) and allocations against that pool. Reuse the SF Compute path only if the customer is buying a block from a provider order book. Routing a job onto NM’s own Ornn contract is a different object. SSH and provisioning stay behind that decision; the strategy note excludes them from v1.
4. **Do not invent stock.** Until the quote is ingested, B300 in the live collector is the retained Nebius card counts and RunPod tiers above, not 56 GPUs in Amsterdam.

## Commands used

Read-only checks on this host. No deploys, no logins to Nebius or GCP, no use of the fleet origin token.

```bash
hostname; uname -a; cat /etc/os-release
ss -tulpn
systemctl list-units --type=service --all
crontab -l
curl -sS -m 5 http://169.254.169.254/latest/meta-data/instance-type
curl -sS -m 5 http://169.254.169.254/latest/meta-data/public-ipv4
getent hosts compute.neural-motion.org compute-preview.neural-motion.org \
  gpu-monitor-origin.neural-motion.org platform.neural-motion.org
curl -sS https://compute.neural-motion.org/api/health
curl -sS https://compute.neural-motion.org/api/public/status
curl -sS https://compute-preview.neural-motion.org/api/health
curl -sS https://compute-preview.neural-motion.org/api/public/status
curl -sS "https://compute-preview.neural-motion.org/api/public/offers"
curl -sS https://nm-compute.pages.dev/api/health
curl -sS -D - -o /tmp/vw.whl https://compute-preview.neural-motion.org/downloads/nmc-0.2.1-py3-none-any.whl
curl -sS https://gpu-capacity-board.pages.dev/data/fleet.json
curl -sk https://gpu-monitor-origin.neural-motion.org/_nm/fleet-snapshot.json
nmc --version
git -C /home/ubuntu/nm-compute status -sb
git -C /home/ubuntu/nm-compute log -1 --oneline
```

Local files read for structure, not for secret values: `/home/ubuntu/nm-compute/README.md`, `docs/ARCHITECTURE.md`, `docs/API.md`, `docs/OPERATIONS.md` (state section and provider-mode table), `worker/src/api/router.js`, `worker/src/api/routes/public.js`, `worker/src/fleet.js`, `worker/src/providers/{all,registry,sfcompute}.js`, `worker/wrangler.core.toml`, `cli/README.md`, `apps/web/src/config/{site,company}.ts`, `apps/web/src/components/brand/family.ts`, `/home/ubuntu/gpu-capacity-board/README.md`, `fleet_publish_config.json`, `data/fleet.json`, `data/latest.json`, `data/canonical.json`, `poll.py` (Nebius `available` = nodes × GPUs).
