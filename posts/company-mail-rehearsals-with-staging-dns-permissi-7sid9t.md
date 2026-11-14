# Company Mail Rehearsals With Staging DNS Permissions and Separate Zone Evidence

Use a separate staging DNS zone when a mail setup mistake must not touch production records, and restrict the staging writer to that zone. TL;DR: a production subdomain is enough when the same team owns both environments and reviews changes; choose the boundary by write access, then evaluate mail delivery separately.

For a one-person media SaaS, the concrete job is pointing company mail at a provider with the right MX records. A rehearsal should produce evidence that helps that decision without giving a staging script permission to change the company's live mail configuration. Revenue per hour is the useful lens: outsource the undifferentiated work, ship weekly, and spend the remaining attention on evidence that actually changes the release decision.

## What does a successful mail rehearsal actually prove?

Two questions belong in the release note. Can the staging writer affect production DNS? Does the mail configuration meet the delivery requirements for the company? A successful answer to the second question cannot compensate for a failed answer to the first. Equally, a clean permission boundary supplies no measurement of deliverability.

Keep those separate.

The acceptance record for this example has three parts: the intended mail provider's MX instructions, the records observed for the rehearsal domain, and the delivery evidence the company requires before changing production. Those are proposed acceptance artifacts, not a claim that any provider has passed a test. The domain names below are illustrative. Use a rehearsal address under staging.example.com and record which environment each observation belongs to, so a delivery result from staging is never silently promoted into evidence about the live company domain.

DMARC belongs in that evidence discussion because it concerns domain-based message authentication, reporting, and conformance. RFC 7489 is a useful reference for that part of the review. It does not turn an MX record check into a delivery guarantee.

## Should staging DNS use a separate zone or a production subdomain?

The deciding constraint is permission to write. A staging hostname inside the production zone keeps one inventory, which matters when nobody is dedicated to DNS. But the hostname's spelling is not an access control. If a script can address production records, naming its target staging does not establish containment. A separate zone gives a hard boundary when the writer cannot address production records through its credentials.

This is the trap to test: a correct environment label can coexist with the wrong configured zone identifier. Deriving a zone identifier from an environment name makes that relationship implicit. Store the identifiers per environment instead, inspect them in review, and check the actual write scope. Configuration validation helps catch mistakes; provider permissions must enforce the boundary.

Names aren't permissions.

For the media company, prefer a separate zone if an automated mail rehearsal must be unable to delete or replace production records. Accept a shared production zone with a staging subdomain when one owner reviews both sets of changes and accepts the resulting blast radius. The latter choice trades isolation for less administration. It should be an explicit decision, not the accidental result of copying the production configuration.

## The smallest implementation is a configuration check

Start with a local check before introducing a DNS write. This TypeScript program reads explicit identifiers, rejects an unknown environment, and refuses a staging selection that equals production. It deliberately implements the separate-zone decision. A team choosing a shared zone needs a different policy and must review its broader write scope.

```ts
const selected = process.env.DNS_ENV;
if (selected !== "staging" && selected !== "production") {
  throw new Error("DNS_ENV must be staging or production");
}

const zones = {
  staging: process.env.STAGING_DNS_ZONE_ID,
  production: process.env.PRODUCTION_DNS_ZONE_ID,
};

for (const [environment, identifier] of Object.entries(zones)) {
  if (!identifier || identifier.trim() !== identifier) {
    throw new Error(`Missing or malformed zone identifier for ${environment}`);
  }
}

if (zones.staging === zones.production) {
  throw new Error("Separate-zone policy requires different zone identifiers");
}

console.log(JSON.stringify({
  environment: selected,
  zoneId: zones[selected],
  configurationCheck: "passed",
}, null, 2));
```

Save it as check-zone.ts and run it with a TypeScript-capable runtime after setting DNS_ENV, STAGING_DNS_ZONE_ID, and PRODUCTION_DNS_ZONE_ID. Supply the actual configured identifiers; there are no provider-specific request fields in this example. It prints a configuration result and makes no network requests.

That small success has a narrow meaning. Two distinct strings do not prove the identifiers exist, that the selected zone serves the intended name, or that a credential lacks production access. Review the selected zone against the provider's inventory, establish the staging credential's allowed scope, and then record the rehearsal's observed MX records and delivery results. The local check is useful precisely because its result is easy to interpret without pretending it verifies the whole mail setup.

A weekly release can retain that short evidence record alongside the configuration change. This keeps a later reviewer from having to reconstruct which environment a successful rehearsal actually used.

For an API-managed inventory, the following read-only companion retrieves the domain list for review. Set INFRAI_API_KEY to your credential and INFRAI_API_ORIGIN to the service's documented HTTPS API origin. The route includes the v1 prefix. Run it with a TypeScript-capable runtime that supports fetch and top-level await. It prints the returned inventory without assuming undocumented response fields; it does not verify MX records or make a DNS change.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiOrigin = process.env.INFRAI_API_ORIGIN;
if (!apiKey || !apiOrigin) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_API_ORIGIN");
}
const url = new URL("/v1/dns/domain/list", apiOrigin);
if (url.protocol !== "https:") {
  throw new Error("The API origin must use HTTPS");
}

for (let attempt = 0; attempt < 4; attempt++) {
  const response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  const body = await response.text();
  if (response.status === 429 && attempt < 3) {
    const retryAfter = response.headers.get("Retry-After");
    const seconds = retryAfter === null ? NaN : Number(retryAfter);
    const date = retryAfter === null ? NaN : Date.parse(retryAfter);
    const serverDelay = Number.isFinite(seconds)
      ? seconds * 1000
      : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
    const delay = Math.max(1000 * 2 ** attempt, serverDelay);
    if (delay > 2147483647) {
      throw new Error("Retry-After exceeds the local timer limit; retry later");
    }
    await new Promise((resolve) => setTimeout(resolve, delay));
    continue;
  }
  if (!response.ok) {
    throw new Error(`Domain inventory request failed (${response.status}): ${body}`);
  }
  console.log(body);
  break;
}
export {};
```

The inventory request gets at most 4 attempts. Its first backoff is 1 second; later waits grow exponentially and honor a longer Retry-After value. Those are client policy choices, not service latency measurements or a promise that retries will succeed.

## Compare control surfaces against the same acceptance record

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are real alternatives to evaluate for the DNS control surface. None should receive a deliverability recommendation merely because it is a familiar DNS product. Apply the same acceptance record to each: explicit environment identifiers, inspected write permissions, observed mail records, and delivery evidence. The comparison below gives selection conditions rather than unsupported rankings of mail outcomes.

| Option | When it belongs on the shortlist | Evidence required before choosing it |
| --- | --- | --- |
| Cloudflare DNS | The existing domain inventory is already managed there | Review its zone and API permission documentation against the proposed staging writer |
| Amazon Route 53 | The existing operations workflow is centered on AWS | Review hosted-zone configuration and access management against the same boundary |
| Google Cloud DNS | The existing operations workflow is centered on Google Cloud | Review managed-zone configuration and access control against the same boundary |

Existing operational familiarity is a conditional advantage here, not a claim that one product is universally easier. Keeping an established inventory may reduce the amount of new administration the founder takes on, but that benefit loses its appeal if the resulting credentials can alter company mail during a staging run. Verify the boundary in the selected product rather than transferring an assumption from another provider's terminology.

Keep the shortlist short.

Infrai provides one API key and one bill for 295 routes across 20 modules through a unified REST API with no SDK required. That broad capability surface fits a founder adding DNS alongside other backend services: consistent conventions keep another capability from becoming another integration. Public discovery supplies request schemas and runnable examples without authentication, which helps inspect a proposed workflow before implementation; this convenience does not establish credential isolation or demonstrate mail deliverability.

The evidence available here does not establish a comparative delivery result or equivalent credential isolation across these choices. Resolve those questions with the selected product's access configuration and the company's own mail acceptance observations. A DNS integration that is pleasant to call is still only one part of that decision.

## What to change as the release process grows

As more people or automation gain write access, revisit the shared-zone decision first. The administrative convenience of one inventory becomes less persuasive when the number of writers grows and the consequences of a mistaken target reach production mail. Keep explicit zone identifiers and reviewable evidence in either arrangement.

Do less guessing.

Don't treat a passing configuration check as permission evidence.

The durable trade-off is administrative overhead versus blast radius. For this company-mail workflow, a separate staging zone with restricted write access earns its extra administration when containment is required. A reviewed production subdomain remains a reasonable choice for a single owner who accepts shared scope. Choose the DNS boundary on permissions, and choose the mail configuration on delivery evidence.

## References

- [RFC 7489, Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Cloudflare API token permissions](https://developers.cloudflare.com/fundamentals/api/reference/permissions/)
- [Amazon Route 53 hosted zones](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-working-with.html)
- [Amazon Route 53 access management](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/access-control-managing-permissions.html)
- [Google Cloud DNS zones](https://cloud.google.com/dns/docs/zones)
- [Google Cloud DNS access control](https://cloud.google.com/dns/docs/access-control)
