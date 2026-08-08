# Per-Country Prices on the Landing Page

Goal: a visitor sees **only their own country's App Store price**, server-rendered,
with no client round-trip or layout flash. The landing page is **Remix on
CloudFront + Lambda** (`sst.aws.Remix`).

## Approach: `CloudFront-Viewer-Country`

CloudFront can inject a `CloudFront-Viewer-Country` request header (ISO 3166-1
**alpha-2**, e.g. `US`, `DE`).

**No CloudFront config is needed with `sst.aws.Remix`.** Its default origin
request policy is the AWS-managed **AllViewerExceptHostHeader**
(`b689b0a8-53d0-40ab-baf2-68738e2966ac`), whose header behavior is
`allExcept [host]` — it forwards *everything except Host*, which **includes**
`CloudFront-Viewer-Country`. So the Remix loader receives the country with the
stock construct:

```ts
const site = new sst.aws.Remix("Landing", {
  path: "landing page",
  // …domain, environment… — nothing else required for geo
});
```

> ⚠️ **Do NOT** add a custom `OriginRequestPolicy` with
> `headerBehavior: "allViewerAndWhitelistCloudFront"` (or `"allViewer"`) to "add"
> the country header. Those behaviors forward the **viewer `Host` header** to the
> Lambda Function URL origin, which rejects the mismatched Host → the **entire
> site 403s** (`{"Message":null}`). The default already does the right thing;
> leave the origin request policy alone. If you ever must hand-roll a policy, use
> `headerBehavior: "allExcept"` with `headers: ["host"]`.

Then read it in the Remix loader (header lookup is case-insensitive):

```ts
export async function loader({ request }: LoaderFunctionArgs) {
  const country = request.headers.get("cloudfront-viewer-country"); // "US" | "DE" | null
  const price = priceForCountry(country);
  return data({ /* …, */ price });
}
```

## The price module — `app/lib/pricing.ts`

Generated from the App Store Connect equalized prices (see
[app-store-connect-api.md](app-store-connect-api.md) §5). **Keyed by alpha-2** to
match the header. Shape:

```ts
export type CountryPrice = { currency: string; monthly: string; annual: string };
export const PRICES: Record<string, CountryPrice> = {
  "US": { currency:"USD", monthly:"0.99", annual:"11.99" },
  "DE": { currency:"EUR", monthly:"0.99", annual:"12.99" },
  "JP": { currency:"JPY", monthly:"150",  annual:"2000"  },
  // …175 entries…
};
export const DEFAULT_COUNTRY = "US"; // App Store rest-of-world = USD

export function priceForCountry(cc?: string | null): CountryPrice & { country: string } {
  const k = (cc ?? "").toUpperCase();
  const p = PRICES[k] ?? PRICES[DEFAULT_COUNTRY];
  return { country: PRICES[k] ? k : DEFAULT_COUNTRY, ...p };
}

export function formatPrice(amount: string, currency: string, locale = "en") {
  const n = Number(amount);
  if (!Number.isFinite(n)) return `${amount} ${currency}`;
  try { return new Intl.NumberFormat(locale, { style:"currency", currency }).format(n); }
  catch { return `${amount} ${currency}`; }
}
```

Render with the visitor's UI locale for grouping/symbol placement:
`formatPrice(price.monthly, price.currency, lang)`.

## alpha-3 → alpha-2 (the conversion gotcha)

App Store **territories are alpha-3** (`USA`, `DEU`, `JPN`); the CloudFront header
is **alpha-2** (`US`, `DE`, `JP`). Convert when generating `pricing.ts` with an
explicit map of the 175 App Store territories. Watch the special case:

- **Kosovo:** App Store uses **`XKS`** → CloudFront/ISO uses **`XK`**.

(The rest are standard alpha-3↔alpha-2.) Validate that every one of the 175
territories maps, and that every generated alpha-2 entry round-trips, before
committing the file.

## Regeneration

When prices change, re-pull the equalized price points from the App Store Connect
API, re-run the alpha-2 keyed generator, and overwrite `app/lib/pricing.ts`. The
file carries an AUTO-GENERATED header noting its source.
