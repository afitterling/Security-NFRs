# App Store Connect REST API — Subscription Setup

Base URL: `https://api.appstoreconnect.apple.com`. Everything below is copy-pasteable
against the live API. Worked example: app `threethings`, bundle `tech.sp33c.three`.

## 0. Auth — ES256 JWT from the `.p8`

You need three values from the App Store Connect API key (Users and Access →
Integrations → App Store Connect API). The same `.p8` EAS uses for submission works
if its role is **Admin** or **App Manager** (needed for pricing writes).

- `KEY_ID` (e.g. `A6A9UP5AGH`)
- `ISSUER_ID` (UUID)
- key file: `~/.appstoreconnect/private_keys/AuthKey_<KEYID>.p8`

JWT shape:
- header: `{ alg: "ES256", kid: KEY_ID, typ: "JWT" }`
- payload: `{ iss: ISSUER_ID, iat: now, exp: now + 600, aud: "appstoreconnect-v1" }`
  — `exp` must be **≤ 20 minutes** out.
- **Sign with `dsaEncoding: "ieee-p1363"`** — Node's default DER encoding produces a
  signature Apple rejects. This is the #1 silent auth failure.

Node helper (no deps):

```js
import crypto from 'node:crypto'; import fs from 'node:fs';
const KEY_ID='…', ISSUER_ID='…';
const KEY_PATH=process.env.HOME+'/.appstoreconnect/private_keys/AuthKey_'+KEY_ID+'.p8';
const b=x=>Buffer.from(x).toString('base64').replace(/=/g,'').replace(/\+/g,'-').replace(/\//g,'_');
function token(){
  const n=Math.floor(Date.now()/1000);
  const si=b(JSON.stringify({alg:'ES256',kid:KEY_ID,typ:'JWT'}))+'.'+
           b(JSON.stringify({iss:ISSUER_ID,iat:n,exp:n+600,aud:'appstoreconnect-v1'}));
  const key=fs.readFileSync(KEY_PATH,'utf8');
  return si+'.'+b(crypto.sign('sha256',Buffer.from(si),{key,dsaEncoding:'ieee-p1363'}));
}
// fetch(BASE+path,{headers:{Authorization:'Bearer '+token()}})
```

## 1. Find the app id

```
GET /v1/apps?limit=200&fields[apps]=name,bundleId,sku
```
Match on `bundleId`. (Prod and dev are separate apps — pick the prod one.)

## 2. Subscription group

```
POST /v1/subscriptionGroups
{ "data": { "type":"subscriptionGroups",
  "attributes": { "referenceName":"premium" },
  "relationships": { "app": { "data": { "type":"apps", "id":"<APP_ID>" } } } } }
```
→ returns the group id.

## 3. Subscriptions (one per period)

```
POST /v1/subscriptions
{ "data": { "type":"subscriptions",
  "attributes": {
    "name":"Premium Monthly",                       // reference name
    "productId":"tech.sp33c.three.premium.monthly",  // PERMANENT, immutable
    "subscriptionPeriod":"ONE_MONTH",                // or ONE_YEAR
    "familySharable": false,
    "groupLevel": 1                                  // same level = crossgrade
  },
  "relationships": { "group": { "data": { "type":"subscriptionGroups", "id":"<GROUP_ID>" } } } } }
```
New subs are born `MISSING_METADATA`. Repeat with `ONE_YEAR` for the annual.

## 4. ⚠️ Availability BEFORE price (critical)

If you `POST /v1/subscriptionPrices` on a sub that has no availability yet, you get:

```
409 ENTITY_ERROR.RELATIONSHIP.INVALID
  source.pointer: /data/relationships/subscriptionPricePoint/id
  detail: "An error occurred while processing the pricing information."
```

This is misleading — the price point is fine; the sub just isn't **available**
anywhere yet. Fix: create availability first. Fetch all territory ids
(`GET /v1/territories?limit=200`, 175 of them), then:

```
POST /v1/subscriptionAvailabilities
{ "data": { "type":"subscriptionAvailabilities",
  "attributes": { "availableInNewTerritories": true },
  "relationships": {
    "subscription": { "data": { "type":"subscriptions", "id":"<SUB_ID>" } },
    "availableTerritories": { "data": [ {"type":"territories","id":"USA"}, … all 175 … ] }
  } } }
```
Note: `GET /v1/subscriptions/<id>/subscriptionAvailability` does **not** accept a
`limit` param, and 404s until you've created availability.

## 5. Price points → prices

Price points are **subscription-scoped and per-territory** (their ids encode the
subscription id + territory + tier; they can rotate, so fetch fresh just before
use). Find the base (DE) price point for your target customer price:

```
GET /v1/subscriptions/<SUB_ID>/pricePoints?filter[territory]=DEU&limit=200
# paginate; match attributes.customerPrice === "0.99"
```

Create the base price:

```
POST /v1/subscriptionPrices
{ "data": { "type":"subscriptionPrices",
  "relationships": {
    "subscription": { "data": { "type":"subscriptions", "id":"<SUB_ID>" } },
    "subscriptionPricePoint": { "data": { "type":"subscriptionPricePoints", "id":"<DE_PP_ID>" } }
  } } }
```

### Equalize to all territories (Apple does NOT auto-do this via API)

Setting the base price prices **only that territory**. To go worldwide, read the
base price point's equalizations (equivalent price point in every other territory)
and create a price for each:

```
GET /v1/subscriptionPricePoints/<DE_PP_ID>/equalizations?include=territory&limit=200
# → ~174 rows: { id (price point), territory, customerPrice, currency }
```
Then `POST /v1/subscriptionPrices` once per row using its price-point id. That's
~174 POSTs per subscription.

Verify coverage: `GET /v1/subscriptions/<id>/prices?include=subscriptionPricePoint,territory`
should return 175 rows (do **not** pass an invalid `fields[...]=territory` — it 400s;
`territory` is a relationship include, not a field).

## 6. Localizations (required to leave MISSING_METADATA)

```
POST /v1/subscriptionLocalizations
{ "data": { "type":"subscriptionLocalizations",
  "attributes": { "name":"Premium Monthly", "locale":"en-US",   // name ≤ 30 chars
                  "description":"Unlock all premium features, billed monthly." },
  "relationships": { "subscription": { "data": { "type":"subscriptions", "id":"<SUB_ID>" } } } } }
```
Plus one group localization:
```
POST /v1/subscriptionGroupLocalizations
{ "data": { "type":"subscriptionGroupLocalizations",
  "attributes": { "name":"Premium", "locale":"en-US" },
  "relationships": { "subscriptionGroup": { "data": { "type":"subscriptionGroups", "id":"<GROUP_ID>" } } } } }
```

## 7. Introductory offer (free trial) — PER TERRITORY

`territory` is a **required** relationship — a single all-territory call 409s with
`ENTITY_ERROR.RELATIONSHIP.REQUIRED`. Loop all 175:

```
POST /v1/subscriptionIntroductoryOffers
{ "data": { "type":"subscriptionIntroductoryOffers",
  "attributes": { "offerMode":"FREE_TRIAL", "duration":"ONE_WEEK", "numberOfPeriods":1 },
  "relationships": {
    "subscription": { "data": { "type":"subscriptions", "id":"<SUB_ID>" } },
    "territory":     { "data": { "type":"territories",  "id":"<TERR>" } }
  } } }
```
To find which territories already have one (e.g. resuming after rate limits):
`GET /v1/subscriptions/<id>/introductoryOffers?limit=200` and read each row's
`relationships.territory.data.id`.

## 8. ⚠️ Rate limiting

Full pricing + trials = ~350 POSTs per sub-pair. The API returns HTTP **429**
("We've received too many requests…") under sustained load. Use **limited
concurrency (~4–8)** and **exponential backoff** with retries; make the loop
**idempotent** (skip territories that already have the price/offer) so you can
re-run to fill gaps.

## 9. Still MISSING_METADATA?

After price + localizations, subscriptions can still report `MISSING_METADATA`
because an **App Review screenshot** is required per subscription. That needs an
image upload flow (reserve `subscriptionAppStoreReviewScreenshots` → PUT bytes to
the upload URL → PATCH `uploaded:true`). Not automated here — supply an image.
