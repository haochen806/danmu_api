# Upstash Redis persistence on Vercel

**Goal:** Make preselected match preferences (`lastSelectMap`) survive on the Vercel
deployment.

**Status:** Done — 2026-08-17. Verified in production.
**Scope:** Configuration only. No code changes were required or made.

---

## 1. Problem

Manual match corrections are saved to `globals.lastSelectMap` (title+season →
animeId/source/offset) and reused by `getPreferAnimeId()` so later matches skip
re-matching. On Vercel they were lost.

Not "never saved" — *inconsistently* saved. Vercel reuses warm instances, so the
map survived some requests, diverged across concurrent instances, and disappeared
when an instance recycled or on every deploy. Symptom: preferences that work for
a while, then silently reset.

## 2. Root cause

Three persistence tiers exist. `danmu_api/apis/dandan-api.js` ~3036–3044:

```js
if (globals.localCacheValid && animeId) writeCacheToFile('lastSelectMap', ...)  // file
if (globals.redisValid      && animeId) setRedisKey('lastSelectMap', ...)       // Upstash REST
if (globals.localRedisValid && animeId) setLocalRedisKey('lastSelectMap', ...)  // local redis
```

On Vercel only the middle one can ever fire:

- `judgeLocalCacheValid()` (`utils/cache-util.js:900`) opens with
  `if (deployPlatform === 'node')` — so `localCacheValid` is never true on Vercel.
- `LOCAL_REDIS_URL` is documented local/Docker-only (`configs/envs.js:681`).
- `ui/template.js:289` states it outright: 云服务部署需要配置redis.

`judgeRedisValid()` (`utils/redis-util.js:279`) only sets `redisValid = true` when
**both** `globals.redisUrl` and `globals.redisToken` are present and a ping succeeds.
With neither set, all three writes were skipped and nothing persisted.

Confirmed at the time: the Vercel project had only `SOURCE_ORDER` and `TOKEN` set.
No Upstash keys anywhere.

## 3. Why Upstash specifically

`utils/redis-util.js` hardcodes the Upstash REST shape — `/ping`, `/get/<key>`,
`/set/<key>`, `/set/<key>?EX=<n>`, `/pipeline`. It is not a Redis client library.
TCP Redis (Redis Cloud, self-hosted) will NOT work without a new util module wired
into all call sites. Serverless can't hold TCP pools anyway.

## 4. What was actually done

Provisioned through the Vercel Marketplace via CLI:

```
vercel integration add upstash/upstash-kv --name danmu-redis -e production -e preview
```

Three things the original plan got wrong or didn't anticipate:

1. **The product slug is `upstash-kv`, not `upstash-redis`.** The latter does not
   exist. Available products: `upstash-qstash`, `upstash-vector`, `upstash-search`,
   `upstash-kv`.
2. **Marketplace terms must be accepted in a browser first.** The CLI returns
   `action_required` / `integration_terms_acceptance_required` with a
   `verification_uri` and cannot complete headlessly. One-time, per team.
3. **The env-var alias gotcha was real.** Vercel injected `KV_REST_API_URL`,
   `KV_REST_API_TOKEN`, `KV_REST_API_READ_ONLY_TOKEN`, `KV_URL`, `REDIS_URL`.
   The code reads none of those. Added, pointing at the same values:

```
UPSTASH_REDIS_REST_URL      <- same value as KV_REST_API_URL
UPSTASH_REDIS_REST_TOKEN    <- same value as KV_REST_API_TOKEN
```

Read at `configs/envs.js:728-729` into `globals.redisUrl` / `globals.redisToken`.
Set for Production and Preview.

Note: both were created as **Sensitive**, so `vercel env pull` returns
`"[SENSITIVE]"` and the API returns `''` even with `decrypt=true`. The values are
recoverable from the `KV_*` vars, which are non-sensitive. `TOKEN` is likewise
sensitive and cannot be read back — testing authenticated endpoints requires
either the real token or a throwaway preview deploy (`vercel deploy --env TOKEN=…`).

## 5. Verification (all performed against production)

`GET /api/config`:

```
redisValid: true
deployPlatform: vercel
```

Cold-start logs show the read path working end to end, with no errors:

```
[system] [redis] 开始发送 PING 请求: …/ping
[system] [redis] getRedisCaches start.
[system] [redis] 开始发送 PIPELINE 请求: …/pipeline
[system] [redis] getRedisCaches completed successfully.
```

Full round trip, driven through the app's own code (match → manual correction on a
different episode → redeploy → cold read):

```
[system] [match] Saving explicit manual preference for "永生" S1
[system] [redis] 开始发送 SET 请求: …/set/lastSelectMap
--- redeploy ---
[system] [redis] Restored lastSelectMap from Redis with 1 entries
```

Persisted value carried the full preference, not just candidates:

```json
"永生": {
  "animeIds": [...],
  "preferBySeason":   {"1": 243251},
  "sourceBySeason":   {"1": "360"},
  "explicitBySeason": {"1": true},
  "offsets":          {"1": "5:【bilibili1】 第9集"}
}
```

Test data was deleted afterwards and the instances cycled.

## 6. Known limitation — corrections can still be lost between warm instances

**Redis fixes loss on instance recycle and on deploy. It does not fix divergence
between concurrently-warm instances.**

During testing, the first manual correction logged
`Saving explicit manual preference` and fired the `SET`, but the value that landed
in Redis had only `animeIds` and no preference fields. An identical correction a
minute later persisted completely.

Likely mechanism: `lastSelectMap` is stored as one whole-map key, and every
instance writes its *entire* in-memory copy. `getRedisCaches()` is guarded by
`globals.redisCacheInitialized`, so an instance reads Redis exactly once at cold
start and never again. A warm instance still holding a pre-correction map can
therefore overwrite a correction another instance just wrote.

Not fully pinned down — the `vercel logs` window was partial, so "no later
overwrite" could not be proven. What is certain: one correction was written and
did not persist intact; a second identical one did.

Supporting evidence for the diagnosis: favorites already have a purpose-built
workaround for exactly this problem — `getFavoriteCachesFromRedis()`
(`utils/redis-util.js:198`) re-reads on every `/favorite` request specifically
because "预热实例可能错过其他实例新增的收藏". `lastSelectMap` has no equivalent.

Cheapest fix if this becomes annoying: re-read `lastSelectMap` from Redis
immediately before a preference write, mirroring the favorites approach.

Also note `[system] [match] Saving explicit manual preference` logs *before* the
save is attempted and does not reflect whether it took. It is not proof on its own.

## 7. Cost — free, and it cannot silently start billing

From the Vercel API for `store_fsnpNJXx6JUC6uKt`:

```json
"billingPlan": { "id": "free", "name": "Free",
                 "paymentMethodRequired": false,
                 "preauthorizationAmount": 0, "cost": "",
                 "details": [{"label": "Monthly Commands", "value": "500000"}] }
"metadata":    { "autoUpgrade": false, "eviction": false }
"usageQuotaExceeded": false
```

- No payment method is attached to the resource — it has no mechanism to bill.
- **`autoUpgrade: false`** is the one that matters: it will not roll onto a paid
  plan as usage approaches 500K commands/month.
- Exceeding the cap **fails closed, not expensive**. `usageQuotaExceeded` flips
  true and commands start erroring; `getRedisCaches` try/catches and `setRedisKey`
  has a `.catch`, so the app logs and degrades to in-memory — i.e. back to the old
  behavior, not broken and not billed.

It can only ever cost money if someone explicitly changes the plan
(`vercel integration update upstash`) or enables auto-upgrade.

Caveats: the free plan allows **1 database per account**, so this is the slot.
`eviction: false` means writes fail rather than evict if the 256 MB ceiling is hit.

Projected usage stays far inside the limits:
- **Reads:** `getRedisCaches()` is guarded by `redisCacheInitialized`, so it runs
  once per instance cold start — one pipeline of 7 GETs.
- **Writes:** `updateRedisCaches()` is hash-guarded (changed keys only).
  `setRedisKey('lastSelectMap')` only on manual correction.
- **Unguarded:** each `/favorite` request does one extra read — deliberate, see §6.
- **Storage:** bounded by `MAX_ANIMES` / `MAX_LAST_SELECT_MAP` with eviction logic.

**Bandwidth is the binding constraint, not commands** — every cold start pulls all
7 keys including the whole `animes` blob. If 10 GB/month is ever approached, lower
`MAX_ANIMES`; it reduces bandwidth and storage together.

## 8. Rollback

Delete `UPSTASH_REDIS_REST_URL` and `UPSTASH_REDIS_REST_TOKEN` and redeploy.
`redisValid` returns to false and behavior reverts to in-memory only. No code or
schema to undo. The Upstash DB can be left in place or deleted.

## 9. Open items

- **The Pi instance may not be persisting either.** No `.cache` directory was found
  at `~/danmu_api` or inside the `danmu-api` container. Check the Pi's `/api/config`
  for `localCacheValid` / `localRedisValid`. If both are false, preference loss
  attributed to Vercel is happening in both places. **Still unchecked.**
- `configs/handlers/vercel-handler.js` can read/write Vercel env vars via the API
  using `DEPLOY_PLATFROM_TOKEN` (typo is in the source). That is for *configuration*
  persistence, not runtime state. Do not use it for `lastSelectMap`.
