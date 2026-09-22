---
title: "App Store Connect Subscriptions via the API"
description: "Create an auto-renewable subscription group, products, localizations, territory availability, prices in every storefront and per-territory free trials with the App Store Connect API. The call order matters (availability before prices), the scripts are idempotent, and it covers what 'Missing Metadata' still needs."
tags: [app-store-connect, storekit, subscriptions, pricing, introductory-offers, spaceship, automation]
related_skills: [07-storekit2-intelligence-based-trial, 20-fastlane-app-store-connect-publishing, 40-app-store-preflight-asc-cli, 64-release-with-remote-ab-experiment]
category: platform-frameworks
---

# App Store Connect Subscriptions via the API

## Context

Setting up a subscription by hand in App Store Connect means dozens of screens: group, products,
localizations, a price per storefront (175 territories), and a free trial per territory. Scripting it makes
it repeatable and reviewable. But the API has ordering rules it doesn't explain, and errors that point at
the wrong cause.

The most expensive lesson: **`POST /v1/subscriptionPrices` fails with "An error occurred while processing
the pricing information" (pointing at `/data/relationships/subscriptionPricePoint/id`) until the
subscription has territory availability.** The error reads like an agreement, banking or tax problem.
It isn't one. Check `Business → Agreements` once, and if the Paid Apps agreement is active, look at call
order instead.

## Pattern

### Order of operations

```
1. subscriptionGroups            (+ subscriptionGroupLocalizations)
2. subscriptions                 (productId, period, groupLevel)
3. subscriptionLocalizations     (display name + description per locale)
4. subscriptionAvailabilities    ← MUST come before prices
5. subscriptionPrices            (one per territory, from the equalized price points)
6. subscriptionIntroductoryOffers (one per territory for a free trial)
7. Review screenshot + attach to an app version (portal; required for the first subscription)
```

### Authentication without a working Bundler setup

Scripts use fastlane's Spaceship client with an API key. If `bundle exec` fails because the project's
Gemfile.lock pins gems that aren't installed, a Homebrew fastlane ships its own gems. Point Ruby at them:

```bash
GEM_HOME="$HOME/.local/share/fastlane/<ruby-version>" \
GEM_PATH="$HOME/.local/share/fastlane/<ruby-version>:$(brew --prefix)/Cellar/fastlane/<version>/libexec" \
$(brew --prefix)/opt/ruby/bin/ruby script.rb
```

(Read the exact paths from the `fastlane` wrapper script: `cat "$(which fastlane)"`.)

### Shared helpers

```ruby
require "spaceship"

Spaceship::ConnectAPI.token = Spaceship::ConnectAPI::Token.create(
  key_id: ENV.fetch("ASC_KEY_ID"), issuer_id: ENV.fetch("ASC_ISSUER_ID"),
  filepath: ENV.fetch("ASC_KEY_PATH"))
C = Spaceship::ConnectAPI.client.tunes_request_client

def rel(type, id) = { data: { type: type, id: id } }

def post(path, type, attributes = {}, relationships = {})
  C.post(path, { data: { type: type, attributes: attributes, relationships: relationships } }).body["data"]
end

# Follows links.next; the API pages at 200.
def all(path, params = {})
  out = []; resp = C.get(path, params.merge(limit: 200))
  loop do
    out.concat(resp.body["data"])
    nxt = resp.body.dig("links", "next") or break
    resp = C.get(nxt.sub(%r{^https://api.appstoreconnect.apple.com/}, ""))
  end
  out
end

APP = Spaceship::ConnectAPI::App.find(ENV.fetch("BUNDLE_ID"))
```

### Steps 1–3: group, products, localizations (idempotent)

```ruby
group = all("v1/apps/#{APP.id}/subscriptionGroups").find { |g| g["attributes"]["referenceName"] == "Pro" } ||
        post("v1/subscriptionGroups", "subscriptionGroups", { referenceName: "Pro" }, { app: rel("apps", APP.id) })

if all("v1/subscriptionGroups/#{group["id"]}/subscriptionGroupLocalizations").empty?
  post("v1/subscriptionGroupLocalizations", "subscriptionGroupLocalizations",
       { locale: "en-US", name: "Pro" }, { subscriptionGroup: rel("subscriptionGroups", group["id"]) })
end

PRODUCTS = [
  # groupLevel 1 = highest tier in the group (drives upgrade/downgrade/crossgrade behaviour).
  { productId: "com.example.app.pro.yearly",  name: "Pro Yearly",  period: "ONE_YEAR",  level: 1, usd: "39.99" },
  { productId: "com.example.app.pro.monthly", name: "Pro Monthly", period: "ONE_MONTH", level: 2, usd: "4.99" },
]
existing = all("v1/subscriptionGroups/#{group["id"]}/subscriptions")
PRODUCTS.each do |p|
  sub = existing.find { |e| e["attributes"]["productId"] == p[:productId] } ||
        post("v1/subscriptions", "subscriptions",
             { name: p[:name], productId: p[:productId], subscriptionPeriod: p[:period],
               groupLevel: p[:level], familySharable: false },
             { group: rel("subscriptionGroups", group["id"]) })
  p[:id] = sub["id"]
  if all("v1/subscriptions/#{sub["id"]}/subscriptionLocalizations").empty?
    post("v1/subscriptionLocalizations", "subscriptionLocalizations",
         { locale: "en-US", name: p[:name], description: "Short benefit line (keep it brief)" },
         { subscription: rel("subscriptions", sub["id"]) })
  end
end
```

Product IDs are permanent: once created (even if never sold) they can't be reused. Decide the naming
scheme before the first `POST`.

### Step 4: availability, before any price

```ruby
territories = all("v1/territories").map { |t| t["id"] }   # 175 at time of writing

PRODUCTS.each do |p|
  begin
    C.get("v1/subscriptions/#{p[:id]}/subscriptionAvailability")   # 404 → not set yet
  rescue
    post("v1/subscriptionAvailabilities", "subscriptionAvailabilities",
         { availableInNewTerritories: true },
         { subscription: rel("subscriptions", p[:id]),
           availableTerritories: { data: territories.map { |t| { type: "territories", id: t } } } })
  end
end
```

### Step 5: prices in every storefront from one base price

Price points are **per subscription**. Pick the base territory's point, then ask for its
**equalizations**, which are Apple's matched price points in every other storefront:

```ruby
PRODUCTS.each do |p|
  base = all("v1/subscriptions/#{p[:id]}/pricePoints", { "filter[territory]" => "USA" })
           .find { |pp| pp["attributes"]["customerPrice"] == p[:usd] } or abort "no #{p[:usd]} point"
  points = { "USA" => base["id"] }
  all("v1/subscriptionPricePoints/#{base["id"]}/equalizations", { include: "territory" }).each do |pp|
    points[pp.dig("relationships", "territory", "data", "id")] = pp["id"]
  end

  priced = all("v1/subscriptions/#{p[:id]}/prices", { include: "territory" })
             .map { |x| x.dig("relationships", "territory", "data", "id") }.compact
  (points.keys - priced).each do |terr|        # idempotent: only what's missing
    post("v1/subscriptionPrices", "subscriptionPrices", {},
         { subscription: rel("subscriptions", p[:id]),
           subscriptionPricePoint: rel("subscriptionPricePoints", points[terr]),
           territory: rel("territories", terr) })
  rescue => e
    warn "#{terr}: #{e.message[0, 120]}"      # log, continue, re-run later
  end
end
```

- Leave out `preserveCurrentPrice` on the first price; it only applies when *changing* a price.
- Omitting `startDate` means "effective immediately".
- That's about 175 POSTs per product, so run it in the background and expect several minutes.

### Step 6: a free trial in every territory

Introductory offers are also per territory:

```ruby
PRODUCTS.each do |p|
  have = all("v1/subscriptions/#{p[:id]}/introductoryOffers", { include: "territory" })
           .map { |o| o.dig("relationships", "territory", "data", "id") }.compact
  (territories - have).each do |terr|
    post("v1/subscriptionIntroductoryOffers", "subscriptionIntroductoryOffers",
         { duration: "ONE_WEEK", offerMode: "FREE_TRIAL", numberOfPeriods: 1 },
         { subscription: rel("subscriptions", p[:id]), territory: rel("territories", terr) })
  rescue => e
    warn "trial #{terr}: #{e.message[0, 120]}"
  end
end
```

Expect a handful of transient `500` responses across about 350 calls. Because every step skips what already
exists, **just run the script again**. The second run only fills the gaps.

### Step 7: what "Missing Metadata" still means

After all of the above, each subscription still shows `MISSING_METADATA` until:

- a **review screenshot** (the paywall showing that product) is uploaded, and
- for the **first** subscription in an app, it's **attached to an app version** and submitted with it
  (it can't be reviewed on its own).

Also check the StoreKit configuration file in the repo matches the new IDs, prices and trials, so local
testing reflects production.

## Edge Cases

- **Changing prices later.** Create new `subscriptionPrices` with a `startDate`. Consider
  `preserveCurrentPrice: true` for existing subscribers.
- **Removing a territory.** Update the availability instead of deleting prices.
- **Other agreements pages.** Apple's EU DAC7 compliance question ("Do any of your apps provide personal
  services?") is about services performed by people, like rides or tutoring. A digital subscription isn't
  one. It doesn't block pricing.

## Why This Matters

A scripted setup is reviewable, repeatable across apps, and safe to re-run after partial failures. Knowing
the availability-first rule avoids hours spent chasing an agreement or tax problem that doesn't exist.

## Anti-Patterns

- ❌ Posting prices before availability (opaque pricing error).
- ❌ Hand-picking a price per storefront instead of using equalizations.
- ❌ Non-idempotent scripts, where a re-run after a 500 creates duplicates or fails on "already exists".
- ❌ Committing API key IDs or key paths. Read them from the environment.
- ✅ Group → products → localizations → availability → prices → trials, each step skipping what exists.
