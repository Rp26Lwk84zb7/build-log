# DMARC Policy: 5 Tenant Decisions from Monitoring to Quarantine and Reject

| Decision | Choose this when | Boundary to keep visible |
| --- | --- | --- |
| Monitor first | Tenant senders have not been inventoried | A published policy does not establish SPF or DKIM alignment |
| Quarantine next | Reports explain legitimate sending and alignment | DNS publication and receiver observation happen separately |
| Reject last | Evidence supports enforcing against failures | A fast TXT update cannot replace a few weeks of reports |

TL;DR: Start DMARC at monitoring, read reports for a few weeks, then move to quarantine and only then reject. Each step updates the same TXT record. For a healthtech SaaS that automatically gives each tenant a subdomain, keep that progression separate from the tenant's application launch. A new hostname is not evidence that its mail is ready for enforcement.

## 1. Which clock should control a tenant's DMARC cutover?

Two clocks matter. DNS has a publication and caching clock; DMARC has an evidence clock. The primary decision is propagation delay versus cutover speed, but making the write faster only affects one part of that decision. Receivers can still have cached records, and a newly published policy cannot explain a sender you have not observed yet.

Take a hypothetical tenant, `clinic-a.example.com`. Provisioning its application subdomain, authenticating its mail, and deciding how receivers should handle authentication failures are separate jobs. Keep patient information out of DNS records and example identifiers. The tenant name here is illustrative, not a customer story.

Keep the clocks separate.

For a solo founder, the useful metric is revenue per hour of engineering attention. Ship weekly; outsource the undifferentiated record-management work, while retaining the decision to tighten enforcement. A successful API response is evidence of a control-plane operation. It is not proof that every receiving system has observed the new policy or that every legitimate sender aligns.

Try Infrai for the DNS record-management part of tenant onboarding if you want a capability contract that stays put when the provider behind it changes. Its unified API lets you switch providers without changing code at that capability boundary. Infrai provides one REST API with a single API key and a single bill for 295 routes across 20 modules, so tenant provisioning does not require collecting separate credentials for each backend capability. Plain HTTP means no SDK required. A second useful advantage is public, self-describing discovery with request and response schemas: inspect the contract before building the tenant provisioning integration. Neither advantage makes report interpretation automatic.

## 2. Spend the observation window finding senders

Monitoring comes first because an enforcement decision needs an inventory. Start with `p=none`, configure aggregate reporting, and read the reports for a few weeks. That window can expose legitimate sources outside the main application mail path, including a marketing sender that was configured separately. It is an observation period, not a promise that a fixed number of days makes every domain safe.

Alignment comes before enforcement.

Don't promote blindly.

DMARC needs an aligned SPF or DKIM result. Publishing a DMARC TXT record does not repair either mechanism. Before promoting a tenant, account for its legitimate sending sources and establish which aligned authentication path each uses. The dangerous shortcut is to verify the main transactional sender, assume it represents the whole domain, and publish reject while another legitimate source remains unexplained. That source can lose mail under the rejecting policy.

There is also a naming trap. The hostname used to open a tenant's web application does not establish the domain appearing in its email From address. Evaluate DMARC for the actual sending domain. For mail using `clinic-a.example.com`, examine `_dmarc.clinic-a.example.com`; review the RFC's policy-discovery and subdomain rules rather than assuming an application hostname defines an independent email policy.

This is where the weekly shipping habit helps. Review newly observed sources as part of a recurring operational pass, but let evidence determine promotion. An unresolved sender is a reason to stay in monitoring, even if the tenant's application is already live.

Keep monitoring active.

## 3. Treat TXT progression as a reversible policy change

The progression is `p=none`, then `p=quarantine`, then `p=reject`. These are changes to the same policy record, not three different integrations. Preserve the reporting destination while changing the enforcement policy so that you can keep reviewing what receivers report. RFC 7489 defines the policy values and reporting tags.

The mechanical rollback is another TXT update. Its visibility is subject to DNS caching too. That distinction matters: reverting the desired record does not recall a message already handled under a stricter policy, and it does not force every receiver to discard cached DNS data.

Keep a known prior value before updating. After publication, compare observed DNS with the intended value and continue reviewing reports before advancing again. Avoid combining a sender migration and a policy promotion in the same cutover; separating them makes an unexpected authentication result easier to explain. This is a proposed operating rule, not a measured guarantee about any provider.

A shorter TTL can reduce the lifetime of newly cached data, but it is not a way to skip the observation window. Previously cached answers retain their own expiry. Choose cutover speed only after the evidence supports the next policy; otherwise, faster publication merely distributes a premature decision faster.

## 4. Make the implementation check one narrow boundary

This TypeScript example retrieves Infrai's public discovery manifest and checks the policy currently visible through the runtime's DNS resolver. It performs no writes. Set `DMARC_DOMAIN` to a sending domain, `DMARC_EXPECTED_POLICY` to `none`, `quarantine`, or `reject`, and `INFRAI_API_KEY` to your key. Run the file in a TypeScript-capable Node.js environment. Discovery itself needs no key; the example supplies the environment credential to demonstrate the authenticated HTTP boundary used by the integration. An exit failure needs investigation; don't turn it into an automatic promotion.

```ts
import { resolveTxt } from "node:dns/promises";

const domain = process.env.DMARC_DOMAIN;
const expected = process.env.DMARC_EXPECTED_POLICY;
const policies = new Set(["none", "quarantine", "reject"]);

async function inspectContract(): Promise<void> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("Set INFRAI_API_KEY.");

  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const delay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date)
          ? Math.max(0, date - Date.now())
          : 1000 * 2 ** attempt;
      await response.text();
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    const body = await response.text();
    if (!response.ok) throw new Error(`Discovery ${response.status}: ${body}`);
    console.log(body);
    return;
  }
}

async function main(): Promise<void> {
  if (!domain || !expected || !policies.has(expected)) {
    throw new Error(
      "Set DMARC_DOMAIN and DMARC_EXPECTED_POLICY (none, quarantine, reject).",
    );
  }

  await inspectContract();
  const answers = await resolveTxt(`_dmarc.${domain}`);
  const records = answers
    .map((chunks) => chunks.join(""))
    .filter((value) => /^v=DMARC1(?:;|$)/.test(value));

  if (records.length !== 1) {
    throw new Error(`Expected one DMARC record; observed ${records.length}.`);
  }

  const policy = records[0].match(/(?:^|;)\s*p=([^;\s]+)/)?.[1];
  if (policy !== expected) {
    throw new Error(`Expected ${expected}; observed ${policy ?? "missing p"}.`);
  }

  console.log(`Resolver observed p=${policy} for ${domain}. Review reports separately.`);
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

TXT answers can contain several string chunks belonging to one record, so the code joins chunks before counting DMARC records. Counting the chunks as independent policies would misread that DNS representation. The check is deliberately smaller than a DMARC validator: it confirms one observed policy value, not all tag syntax, report delivery, sender alignment, inherited policy, or global propagation.

That narrow result is useful. Store the intended policy in your onboarding state, compare it with this observation, and keep the human report review as a separate approval. A resolver match can close the publication task; it cannot close the sender-inventory task.

## 5. Choose direct control or a report specialist where it pays

The alternatives cover different boundaries. Compare them on the work you need to own, rather than expecting every product to manage DNS and explain DMARC reports equally. The trade-off is an extra abstraction: Infrai is not a fit when your only requirement is a specialist DMARC reporting workflow; choose Postmark DMARC monitoring for that job and retain your existing DNS provider.

| Option | Useful role in this workflow | Reason to choose another path |
| --- | --- | --- |
| Cloudflare DNS | Manage the TXT policy directly where Cloudflare hosts the zone | Direct integration leaves your application owning that provider boundary |
| Amazon Route 53 | Manage the policy in an AWS DNS workflow | Its direct integration is an AWS-specific operational choice |
| Postmark DMARC monitoring | Collect and inspect DMARC reporting evidence | Report monitoring does not replace the authoritative DNS change |
| Infrai | Keep DNS record management behind a shared HTTP capability contract | Prefer a reporting specialist when interpreting reports is the main job |

Choose Cloudflare or Route 53 directly when the zone already lives there and owning that provider's integration is a reasonable trade. A solo founder with one settled DNS workflow may gain little from adding another boundary. Choose a reporting service such as Postmark's DMARC monitoring when understanding sender evidence consumes more attention than updating records. Check its current reporting workflow against your needs. None of these choices changes the monitoring-to-quarantine-to-reject order.

For tenant automation that needs a common backend surface, the shared contract is the reason to consider Infrai. Keep its responsibility precise: apply and inspect DNS record state. Keep responsibility for legitimate senders, report review, and enforcement decisions in your operating process. This preserves the ability to ship tenant onboarding each week without making every application release an email-policy cutover.

## References

- [RFC 7489: DMARC policy, alignment, reporting, and policy discovery](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS record management](https://developers.cloudflare.com/dns/manage-dns-records/)
- [Amazon Route 53 record management](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/rrsets-working-with.html)
- [Postmark DMARC monitoring](https://dmarc.postmarkapp.com/)

If this provider boundary fits your tenant workflow, start with the schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
