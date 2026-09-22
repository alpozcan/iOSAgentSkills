---
title: "Sentry Crash Analysis for Releases and Experiments"
description: "Read-only, scriptable crash analysis over the Sentry REST API: release health, unhandled and watchdog (OOM) terminations, regression checks between releases with confidence intervals, single-event forensics from breadcrumbs, and crash comparison by experiment arm, including the traps (regional hosts, sessions can't group by custom tags, endpoint-specific period limits)."
tags: [sentry, crash-analysis, release-health, oom, watchdog, statistics, experiments, rest-api]
related_skills: [32-sentry-telemetrydeck-integration, 39-structured-logging-crash-analytics, 25-performance-monitoring-and-profiling-patterns, 64-release-with-remote-ab-experiment]
category: testing-and-debugging
---

# Sentry Crash Analysis for Releases and Experiments

## Context

Setting up Sentry is covered elsewhere ([[32-sentry-telemetrydeck-integration]]). This skill is about
**reading** it programmatically after a release: "did X.Y regress?", "what is this fatal event?", "does the
experiment's treatment arm crash more?". An MCP server for Sentry may be unavailable or return HTML. The
REST API with a read-only token always works, and a small stdlib script turns it into repeatable answers.

## Pattern

### Setup: token, region, identifiers

```bash
export SENTRY_AUTH_TOKEN=...          # scopes: org:read project:read event:read (no write needed)
export SENTRY_ORG=my-org SENTRY_PROJECT=my-project
# Find the project's numeric id and the org's REGION host (EU orgs live on de.sentry.io):
curl -s -H "Authorization: Bearer $SENTRY_AUTH_TOKEN" https://sentry.io/api/0/organizations/$SENTRY_ORG/ \
  | python3 -c 'import json,sys; d=json.load(sys.stdin); print(d["links"]["regionUrl"], d["access"])'
curl -s -H "Authorization: Bearer $SENTRY_AUTH_TOKEN" https://sentry.io/api/0/organizations/$SENTRY_ORG/projects/ \
  | python3 -c 'import json,sys; [print(p["slug"], p["id"]) for p in json.load(sys.stdin)]'
export SENTRY_BASE=https://de.sentry.io/api/0 SENTRY_PROJECT_ID=123456
```

Use the org's `regionUrl` as the base. The global host answers some calls but not all.

Release names from the Cocoa SDK look like `<bundle-id>@<version>+<build>`. Let the script accept
`X.Y.Z+N` and add the prefix.

### Helpers (Python stdlib only)

```python
import json, math, os, re, urllib.parse, urllib.request

BASE, ORG = os.environ["SENTRY_BASE"], os.environ["SENTRY_ORG"]
PROJ, PID = os.environ["SENTRY_PROJECT"], os.environ["SENTRY_PROJECT_ID"]
HDR = {"Authorization": f"Bearer {os.environ['SENTRY_AUTH_TOKEN']}"}

def get(path, params=None):
    url = BASE + path + ("?" + urllib.parse.urlencode(params, doseq=True) if params else "")
    with urllib.request.urlopen(urllib.request.Request(url, headers=HDR), timeout=60) as r:
        return json.load(r), r.headers.get("Link", "")

def paginate(path, params, limit=None):
    rows, link = get(path, params); rows = list(rows)
    while (limit is None or len(rows) < limit) and 'rel="next"; results="true"' in link:
        cur = re.search(r'rel="next"; results="true"; cursor="([^"]+)"', link).group(1)
        more, link = get(path, {**params, "cursor": cur}); rows += more
    return rows[:limit] if limit else rows

def discover(fields, query="", period="14d"):
    p = {"project": PID, "field": fields, "statsPeriod": period, "dataset": "errors", "per_page": 100, "sort": "-count()"}
    if query: p["query"] = query
    return get(f"/organizations/{ORG}/events/", p)[0].get("data", [])

def wilson(k, n, z=1.96):                      # 95% CI for a small-sample proportion
    if n == 0: return (None, None)
    p = k / n; d = 1 + z*z/n; c = (p + z*z/(2*n)) / d
    h = z * math.sqrt(p*(1-p)/n + z*z/(4*n*n)) / d
    return max(0, c - h), min(1, c + h)

def fisher(a, b, c, d):                        # two-sided exact test for [[a,b],[c,d]]
    n1, n2, k = a+b, c+d, a+c; n = n1+n2
    lp = lambda x: (math.lgamma(n1+1)-math.lgamma(x+1)-math.lgamma(n1-x+1)+math.lgamma(n2+1)
                    -math.lgamma(k-x+1)-math.lgamma(n2-k+x+1)-math.lgamma(n+1)+math.lgamma(k+1)+math.lgamma(n-k+1))
    obs = lp(a)
    return min(1.0, sum(math.exp(lp(x)) for x in range(max(0, k-n2), min(k, n1)+1) if lp(x) <= obs + 1e-9))
```

### Release health per release

```python
data, _ = get(f"/organizations/{ORG}/sessions/", {
    "project": PID, "groupBy": "release", "statsPeriod": "14d", "interval": "1d",
    "field": ["sum(session)", "count_unique(user)", "crash_free_rate(session)", "crash_free_rate(user)"]})
for g in data["groups"]: print(g["by"]["release"], g["totals"])
```

`crash_free_rate(...)` can't be combined with `groupBy=session.status`, so ask for them separately.

### Fatal and unhandled breakdown, including OOM

```python
discover(["release", "error.type", "error.mechanism", "device.class", "count()", "count_unique(user)"],
         "error.handled:false", "30d")
```

How memory deaths show up in an iOS app:

| Signal | In Sentry |
|---|---|
| Watchdog / jetsam kill | `error.type:WatchdogTermination`, `error.mechanism:watchdog_termination`, level fatal, **reported on the next launch** (so it lags) |
| Allocation failure | `C++ Exception: St9bad_alloc`, mechanism `cpp_exception`, often no crashed thread and no stack |
| Pressure beforehand | breadcrumbs such as "Memory warning" / "Low memory", often several in the last second |
| Headroom at the time | `contexts.device.memory_size`, `usable_memory`, `free_memory` |

An id in an alert email may be a **user** id, not an issue or event id. Resolve it by searching
`user.id:<id>` in Discover.

### Regression check between two releases

```python
def crashed_users(release, period="30d"):
    r = discover(["count_unique(user)"], f"error.handled:false release:{release}", period)
    return r[0]["count_unique(user)"] if r else 0

# users per release from /sessions/ above; then:
p = fisher(t_crashed, t_users - t_crashed, b_crashed, b_users - b_crashed)
print(wilson(b_crashed, b_users), wilson(t_crashed, t_users), p)

# New issues introduced by the target release. The issues endpoint only accepts statsPeriod "", "24h" or "14d":
new = paginate(f"/projects/{ORG}/{PROJ}/issues/",
               {"query": f"is:unresolved first-release:{target}", "statsPeriod": ""}, limit=50)
```

Report the interval and the p-value, not just "crash-free went from 98.9% to 99.6%" (illustrative). In a small sample
(hundreds of users), one crashed user is noise.

### Single-event forensics

```python
e, _ = get(f"/projects/{ORG}/{PROJ}/events/{event_id}/")
tags = {t["key"]: t["value"] for t in e["tags"]}
device = e["contexts"].get("device", {})
for entry in e["entries"]:
    if entry["type"] == "breadcrumbs":
        for c in entry["data"]["values"][-25:]:
            print(c.get("timestamp", "")[11:19], c.get("category"), (c.get("message") or c.get("data"))) 
```

The last 20–30 breadcrumbs usually tell the story: which screen, how many background/foreground cycles,
and how many memory warnings came right before the kill.

### Crashes by experiment arm

If the app tags its scope with the arm (see [[64-release-with-remote-ab-experiment]]):

```python
events  = discover(["arm_tag", "count()", "count_unique(user)"], "has:arm_tag")
crashes = discover(["arm_tag", "error.mechanism", "count_unique(user)"], "has:arm_tag error.handled:false")
```

**Sentry's release-health sessions can't be grouped by a custom tag**, so there is no per-arm crash-free
rate in Sentry. Take the per-arm **denominator** (exposed users) from the experiment platform, and use
Sentry for the numerator only:

```python
rate_t, rate_c = crashed_t / exposed_t, crashed_c / exposed_c
p = fisher(crashed_t, exposed_t - crashed_t, crashed_c, exposed_c - crashed_c)
```

Early in a rollout, "no events carry the tag" is normal. Then check that the tag is actually set before
the first risky code runs.

### Package it

Wrap these as subcommands (`releases`, `crashes`, `issues`, `compare`, `event`, `arms`), each with a
table output and `--json` for machine use. Store it as an agent skill, so "check crashes for X.Y" becomes
one call instead of a session of ad-hoc curl.

## Edge Cases

- **HTTP 400 "Invalid stats_period"** on `/issues/` → use `24h`, `14d` or `""`.
- **Empty tag values** (`/tags/<key>/values/` returns `[]`) → no event has carried the tag yet. It's not
  an error.
- **Crash-free vs. crashed users.** Sentry's crash-free rate counts crashed *sessions/users* as the SDK
  reports them. "Unhandled events" in Discover can differ slightly; say which one you're quoting.
- **Tokens.** Read the token from the environment. Never echo it; log at most its length.

## Why This Matters

A post-release check becomes a two-minute, reproducible command with honest statistics. It also stops
over-confident conclusions from tiny samples, which matter most exactly when a release or experiment is
small.

## Anti-Patterns

- ❌ Using the global host for an EU-region org and trusting partial results.
- ❌ A per-arm crash rate with Sentry sessions as the denominator.
- ❌ Declaring a regression or an improvement without an interval.
- ❌ Treating an alert's user id as an issue id.
- ✅ Region-correct base, Discover for numerators, the experiment platform for denominators, Wilson and
  Fisher for small counts.
