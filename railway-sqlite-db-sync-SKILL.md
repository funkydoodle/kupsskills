---
name: railway-sqlite-db-sync
description: >
  Use when two Railway environments (staging/production) have diverged SQLite tables and
  need to be synced. Covers adding temporary export/import JSON endpoints, deploying via
  Railway GraphQL API, performing the sync via curl + Python diff, then reverting both
  environments to the original commit.
---

# Railway SQLite DB Sync

Sync a SQLite table between two Railway environments when direct volume/SSH access is unavailable. The pattern: add temporary JSON endpoints → deploy → sync via HTTP → revert.

---

## When to Use

- Two Railway environments have diverged data in a SQLite table
- Railway SSH requires browser auth (non-interactive)
- No direct access to the `.db` file (Railway volumes are remote)

---

## Step 1 — Add Temporary Export/Import Endpoints

Add two auth-gated routes to your admin router. These should sit inside your existing authenticated middleware scope.

```ts
// GET /admin/<resource>/export — returns all rows as JSON
app.get('/<resource>/export', async (_request, reply) => {
  const rows = getDb().select().from(yourTable).all();
  return reply.send({ rows, total: rows.length });
});

// POST /admin/<resource>/import — inserts missing rows (no overwrites)
app.post('/<resource>/import', async (request, reply) => {
  const { rows: incoming } = request.body as { rows?: Record<string, unknown>[] };
  if (!Array.isArray(incoming)) return reply.status(400).send({ error: 'rows array required' });

  let inserted = 0, skipped = 0;
  for (const row of incoming) {
    if (!row.address) { skipped++; continue; }  // swap for your unique key
    try {
      getDb().insert(yourTable).values({ ...row }).onConflictDoNothing().run();
      inserted++;
    } catch { skipped++; }
  }

  return reply.send({ inserted, skipped, total: incoming.length });
});
```

**Key rules:**
- `onConflictDoNothing()` — never overwrite existing rows
- Auth is inherited from existing middleware — no extra auth logic needed
- Remove these endpoints after the sync is complete

Commit and push to your sync branch. Get the full commit SHA:
```bash
git push origin <branch> && git rev-parse HEAD
```

---

## Step 2 — Deploy to Both Environments via Railway GraphQL API

Get your Railway API token from project settings, then deploy the new commit:

```bash
TOKEN="<railway-project-token>"
NEW_SHA="<new-commit-sha>"
SERVICE_ID="<service-id>"

# Deploy to each environment
curl -s -X POST https://backboard.railway.app/graphql/v2 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"query\": \"mutation { serviceInstanceDeployV2(commitSha: \\\"$NEW_SHA\\\", environmentId: \\\"$STAGING_ENV_ID\\\", serviceId: \\\"$SERVICE_ID\\\") }\"}"
```

Repeat for each environment ID. Capture the returned deployment IDs.

**Poll until both are SUCCESS:**
```bash
curl -s -X POST https://backboard.railway.app/graphql/v2 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"query\": \"query { deployment(id: \\\"$DEPLOY_ID\\\") { status } }\"}"
```

Poll every 15s. Wait for `SUCCESS` before proceeding — typically takes 2–4 minutes.

---

## Step 3 — Login and Perform the Sync

```bash
# Get session cookies (adjust field name and password to your admin login)
STAGING_COOKIE=$(curl -s -c - -X POST https://<staging-url>/admin/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "password=<admin-password>" -L | grep session | awk '{print $6"="$7}' | head -1)

PROD_COOKIE=$(curl -s -c - -X POST https://<prod-url>/admin/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "password=<admin-password>" -L | grep session | awk '{print $6"="$7}' | head -1)

# Export from both
curl -s -b "$STAGING_COOKIE" https://<staging-url>/admin/<resource>/export -o /tmp/staging.json
curl -s -b "$PROD_COOKIE"    https://<prod-url>/admin/<resource>/export    -o /tmp/prod.json
```

**Compute diff and import missing rows** (swap `address` for your unique key):
```python
import json

staging = json.load(open('/tmp/staging.json'))['rows']   # adjust key to match export shape
prod    = json.load(open('/tmp/prod.json'))['rows']

staging_keys = {r['address'] for r in staging}
prod_keys    = {r['address'] for r in prod}

to_prod    = [r for r in staging if r['address'] not in prod_keys]
to_staging = [r for r in prod    if r['address'] not in staging_keys]

print(f"Staging: {len(staging)}, Prod: {len(prod)}")
print(f"→ prod needs {len(to_prod)}, staging needs {len(to_staging)}")

json.dump({'rows': to_prod},    open('/tmp/to-prod.json', 'w'))
json.dump({'rows': to_staging}, open('/tmp/to-staging.json', 'w'))
```

```bash
# Import missing rows to each env
curl -s -X POST -b "$PROD_COOKIE" -H "Content-Type: application/json" \
  -d @/tmp/to-prod.json https://<prod-url>/admin/<resource>/import

curl -s -X POST -b "$STAGING_COOKIE" -H "Content-Type: application/json" \
  -d @/tmp/to-staging.json https://<staging-url>/admin/<resource>/import
```

**Verify counts match:**
```bash
STAGING_N=$(curl -s -b "$STAGING_COOKIE" https://<staging-url>/admin/<resource>/export | python3 -c "import sys,json; print(json.load(sys.stdin)['total'])")
PROD_N=$(curl -s -b "$PROD_COOKIE" https://<prod-url>/admin/<resource>/export | python3 -c "import sys,json; print(json.load(sys.stdin)['total'])")
[ "$STAGING_N" = "$PROD_N" ] && echo "MATCH: $STAGING_N" || echo "MISMATCH: staging=$STAGING_N prod=$PROD_N"
```

---

## Step 4 — Revert Both Environments to Original Commit

Re-deploy the pre-sync commit SHA using the same GraphQL mutation:

```bash
ORIGINAL_SHA="<original-commit-sha>"

# Deploy original to staging
curl -s -X POST https://backboard.railway.app/graphql/v2 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"query\": \"mutation { serviceInstanceDeployV2(commitSha: \\\"$ORIGINAL_SHA\\\", environmentId: \\\"$STAGING_ENV_ID\\\", serviceId: \\\"$SERVICE_ID\\\") }\"}"

# Deploy original to production
curl -s -X POST https://backboard.railway.app/graphql/v2 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"query\": \"mutation { serviceInstanceDeployV2(commitSha: \\\"$ORIGINAL_SHA\\\", environmentId: \\\"$PROD_ENV_ID\\\", serviceId: \\\"$SERVICE_ID\\\") }\"}"
```

Poll both revert deployment IDs until `SUCCESS`.

---

## Quick Reference

| What you need | Where to get it |
|---|---|
| Railway API token | Railway dashboard → Project Settings → Tokens |
| Service ID | Railway dashboard → Service → Settings → Service ID |
| Environment IDs | Railway GraphQL: `query { project(id: "...") { environments { edges { node { id name } } } } }` |
| Original commit SHA | `git rev-parse HEAD` before creating sync branch |

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Export endpoint uses wrong JSON key | Make sure export key (`rows`) matches what the diff script reads |
| Forgot `onConflictDoNothing()` | Without it, re-running import overwrites existing data |
| Reverting before verifying counts | Always confirm counts match before reverting |
| Polling too infrequently | Railway builds typically take 2–4 min; poll every 15s |
| Cookie grep fails on some curl versions | Use `-c /tmp/cookies.txt` + `-b /tmp/cookies.txt` file-based cookies as fallback |
