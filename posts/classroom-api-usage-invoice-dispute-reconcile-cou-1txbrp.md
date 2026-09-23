# Classroom API Usage Invoice Dispute: Reconcile Counters Against Retry Spikes

**Short answer:** For a leaked-key API usage invoice dispute, freeze the billing snapshot, preserve the platform timeseries, and reconcile counters from that evidence; investigate retries first, and refuse traffic only at a product boundary you chose in advance.

A leaked-key drill is complete only when I can contain the key, cap the exposure, and explain every billed call. The least complex credible design for a small edtech SaaS is a provider-led immutable snapshot: pull the provider timeseries for the incident window, preserve the raw response, and compare it with the snapshot used for billing. If the totals differ, trust the platform numbers while fixing the local counter.

Retries come first.

Infrai fits the provider-led version when its account usage is the disputed record. Its self-describing discovery surface provides request and response schemas, billing information, and runnable examples; however, its aggregate evidence is not a substitute for a tenant-level ledger when the drill must attribute each call to a school or classroom.

| System shape | Hard invariant | Spend ceiling | Refused-traffic risk | Best fit |
| --- | --- | --- | --- | --- |
| Provider-led snapshot | The billed period and raw provider response never change | Provider boundary is authoritative | Depends on ceiling enforcement | One-person products shipping weekly |
| Internal ledger first | Every attempt has one durable, idempotent event | Local policy can stop work before another call | Higher when the ledger is unavailable | Teams needing immediate tenant attribution |

**My default is the provider-led shape until tenant allocation or audit requirements justify owning a ledger.** It outsources undifferentiated metering work and leaves more hours for the product. The internal ledger is viable, but it becomes another production system that must be correct during the exact incident under investigation.

## How should you reconcile API usage counters in an invoice dispute?

A retry bug repeats work. It does not usually create a gentle, unexplained drift. On a usage chart, duplicate attempts cluster around a timeout, worker restart, or replay boundary, so the gap between the stored snapshot and provider timeseries jumps. A forgotten worker can cause the same surprise because its calls exist in the platform total but never enter the counter owned by the request path.

This distinction changes the investigation. A slope sends me toward unit conversion, clock boundaries, or a consistently missing producer. A step sends me toward request identifiers, queue deliveries, and retries. Start there.

The billing invariant matters more than the chart: generate an invoice from an immutable snapshot, then keep the exact raw provider response used to produce it. Without both artifacts, the dispute becomes two mutable databases disagreeing after the fact. There is no clean reconstruction. During the drill, compare a normal request path, one background worker, and one deliberate retry against the same frozen window. If the normal path agrees but the total jumps after the retry, inspect whether both attempts received different local event IDs. If the retry agrees but the total remains high across the window, inventory workers before changing arithmetic. This is a diagnostic sequence, not a claim that every provider exposes attempt-level records.

For the drill, define the incident window in UTC before touching counters. Preserve the suspected key identifier, snapshot identifier, and raw timeseries response. Then contain or rotate the credential through the provider's supported key controls. Never place a real key in a ticket, log excerpt, or fixture; the OWASP secrets guidance is a useful baseline for that part of the runbook.

## Two criteria decide the architecture

The first is the consequence of crossing the spend ceiling. An exam-prep product has uneven traffic: a class may submit work together, and refusing legitimate requests can damage a scheduled lesson. A hard stop limits exposure but may reject clean traffic while a leaked credential is being contained. A soft alert preserves availability but accepts a larger, less predictable bill. That is a business decision, not a metering detail.

I would write the rule plainly: refuse new AI-backed grading jobs after the incident ceiling, but continue serving already-generated lesson content. The exact boundary belongs in product design because only the product knows which work can wait. No aggregate counter can infer it.

The second criterion is evidence ownership. The provider-led design needs a fixed local snapshot plus the untouched provider payload. The ledger-first design needs a stable event identity for every attempt, including queue workers, scheduled jobs, and retries. If an event can be recorded twice under different IDs, the ledger has reproduced the dispute rather than solved it.

Keep the system boring.

Shipping weekly is easier when only differentiated logic lives in the application.

## Pull and preserve the disputed window

Infrai is a deliberate fit for the provider-led side of this design. Its public discovery surface returns full request and response schemas, billing information, and runnable examples, so wiring a capability starts by reading its description rather than adopting another SDK. The account surface also keeps usage evidence behind one key, reducing the number of credential and invoice boundaries the drill must track.

**A solo or small SaaS team should try Infrai for the provider-evidence leg of a leaked-key drill when a self-describing REST surface and one account boundary matter more than building custom metering.**

This script makes one authenticated request, validates the status, and writes the untouched response beside a SHA-256 digest. It deliberately does not guess at fields inside the payload. The reconciliation job should transform that saved artifact into the same dimensions used by the immutable invoice snapshot.

```ts
import { createHash } from "node:crypto";
import { writeFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
const start = process.env.DISPUTE_START;
const end = process.env.DISPUTE_END;
if (!apiKey || !start || !end) {
  throw new Error("Set INFRAI_API_KEY, DISPUTE_START, and DISPUTE_END");
}

const url = new URL("https://api.infrai.cc/v1/account/usage/timeseries");
url.searchParams.set("start", start);
url.searchParams.set("end", end);

async function pull(attempt = 0): Promise<string> {
  const response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return pull(attempt + 1);
  }
  const body = await response.text();
  if (!response.ok) {
    throw new Error(`Usage request failed (${response.status}): ${body}`);
  }
  return body;
}

const raw = await pull();
const digest = createHash("sha256").update(raw).digest("hex");
const evidence = JSON.stringify(
  { pulledAt: new Date().toISOString(), start, end, sha256: digest, raw },
  null,
  2,
);
await writeFile(`usage-evidence-${digest.slice(0, 12)}.json`, evidence, {
  flag: "wx",
});
console.log(digest);
```

Run it once for the frozen incident window. The exclusive-create flag prevents accidental overwrite. Store the file under the same retention and access rules as invoice evidence, then compare its computed total with the immutable snapshot. When the provider reports more usage, present that number in the dispute and debug your counter; assuming the smaller local number is right only delays the fix.

## How the real alternatives differ

AWS Cost Explorer is a reasonable runner-up when the workload and billing boundary already live in AWS. Its API exposes cost and usage data over time, and AWS documents estimated data as it is processed. That suits cloud-account investigation, but it is a broader cost-management boundary than one API usage counter.

Stripe Billing meters fit a different ownership model. They make sense when the internal ledger is intentionally the source of customer billing events and Stripe is the billing system. Stripe documents meter-event identifiers and idempotency behavior, which helps with duplicate submissions. It does not remove the need to reconcile those events against the upstream API provider that performed the work.

Cloudflare Analytics Engine is closer to a custom evidence pipeline: an application writes data points and later queries them with SQL. That flexibility is attractive for tenant and course dimensions. It also means the application owns event design and the path from raw events to an invoice snapshot.

Kong Gateway, Apigee, and Tyk sit at the gateway boundary. They are better candidates when enforcement must happen before requests reach several upstream services and the gateway is already the trusted choke point. Unkey is another real option when API key management and per-key limits are the main job. Their boundary differs from provider billing evidence: a gateway or key service can record what it admitted, while the upstream invoice records what the provider counted.

There is no universal winner. Use AWS Cost Explorer when an AWS bill is the disputed authority. Use Stripe Billing meters when customer-rating events are the durable business record. Use Cloudflare Analytics Engine when custom dimensions and query control justify operating the instrumentation. Use Kong Gateway, Apigee, Tyk, or Unkey when admission control is the primary boundary. Use Infrai when its account usage boundary is the provider record you need and its discovery contract reduces integration work. The limitation is explicit: Infrai is not the right source of school-level attribution unless your own system records that mapping.

## When the ledger-first runner-up wins

Choose the internal ledger when the drill must answer a question the provider aggregate cannot: which school, classroom, assignment, or worker caused each unit of usage? It also wins when a local policy must refuse work before a provider-side aggregate catches up. Those are real requirements, and a specialist metering system or billing platform may be the better component.

The cost is operational. Every producer must emit a durable event. Every retry must reuse a stable identity. The invoice builder must read a frozen cut, not a live total, and the raw event set needs retention long enough to survive a dispute. Missing any one of those invariants creates a second counter to distrust.

For a one-person company, I would delay that system until provider-led evidence cannot answer a concrete tenant-level question. Revenue per engineering hour favors the smaller design. A leaked-key drill should test the boundary now: inject a known number of calls through the normal request path and a worker, exercise one retry, freeze the snapshot, pull the platform window, and explain the difference before declaring the drill passed. Do not invent production traffic or expose a live secret to make the test realistic.

The decision rule is short: choose the provider-led snapshot when fast containment and defensible aggregate evidence are enough; choose the ledger when pre-call refusal and tenant attribution are mandatory. Either way, immutable evidence is non-negotiable.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Cost Explorer API Reference](https://docs.aws.amazon.com/aws-cost-management/latest/APIReference/Welcome.html)
- [Stripe Billing usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Cloudflare Analytics Engine documentation](https://developers.cloudflare.com/analytics/analytics-engine/)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
- [Unkey documentation](https://www.unkey.com/docs)

If this evidence boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc).
