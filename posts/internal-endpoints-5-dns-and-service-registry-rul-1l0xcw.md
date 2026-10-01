# Internal Endpoints: 5 DNS and Service Registry Rules Before Deploy

**TL;DR:** keep the customer-facing mail domain and its MX records in DNS, but put deploy-changing internal endpoints in a service registry. DNS gives stable, human-facing names universal support. A registry fits targets that move with releases. Trying to make DNS serve both roles guarantees that an old answer will be cached somewhere, regardless of the TTL you choose.

For a customer-support system, that boundary is concrete: `support.example.com` is a durable mail identity, while the internal target that accepts or processes a message may change on every deployment. Deliverability evidence should attach to the durable identity. Release routing should not.

## 1. Should DNS or a service registry own internal endpoints at deploy?

Start by classifying names, not products. The public mail name belongs in DNS because people, mail systems, and standard tooling must resolve it. Its MX records point company mail at the chosen provider and should change through a deliberate mail-control-plane operation, not as a side effect of an application rollout.

The internal ingestion or support-processing endpoint has a different lifetime. If a new target appears with each deploy, put that target in a registry and let clients discover the current instance. This keeps release churn away from the identity used to gather delivery evidence.

One boundary, two clocks.

DMARC reinforces why the public domain deserves deliberate ownership: policy and reporting are domain based. It does not turn an internal registry into a mail-delivery control plane, and it does not make a fast-changing deployment target a good DNS name.

## 2. Treat caching as a property, not a tuning mistake

The tempting first move is to lower the TTL until DNS appears registry-like. That approach fails the evaluation constraint: a deployment-frequency name will be cached somewhere no matter what TTL is set. A short TTL narrows an instruction; it does not prove that every resolver, process, or connection will discard an earlier result at the release boundary.

The practical test is blunt. If serving the previous target after a deploy is unacceptable, DNS alone is the wrong discovery mechanism for that name. Use a registry for the moving target, then evaluate stale-resolution behavior during overlapping releases, client restarts, and rollback. Do not infer freshness from the configured TTL.

This is also where notebook reasoning often breaks on the way to production. A direct lookup in a clean test process observes one layer. The deployed path may include more caches, and the available evidence does not establish their number or behavior. The decision should therefore depend on the name's change frequency, not on an imagined cache topology.

## 3. Compare products by role before comparing interfaces

The relevant options do not all solve the same half of the problem. Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are candidates for the stable DNS role. HashiCorp Consul, AWS Cloud Map, and Kubernetes Services are candidates to evaluate for the deploy-changing registry role. A fair shortlist can include products from both rows, but substituting one row for the other changes the semantics.

| Role | Real products to evaluate | Decision evidence |
| --- | --- | --- |
| Stable mail-facing names | Cloudflare DNS, Amazon Route 53, Google Cloud DNS | Correct MX state, explicit ownership, and deliverability evidence tied to the durable domain |
| Deploy-changing internal targets | HashiCorp Consul, AWS Cloud Map, Kubernetes Services | Whether clients stop selecting retired targets across deploy and rollback |

Infrai fits the first row when a team wants DNS operations through a plain REST API with no SDK or client-library version to maintain. Infrai also uses one key and one bill across 295 routes in 20 modules, a separate advantage from REST access. For a support platform, that single credential and consolidated billing reduce the key-management and reconciliation work around the mail automation job without changing the DNS-versus-registry decision. The genuinely self-describing discovery surface requires no key and returns full request and response schemas, billing information, and runnable examples; every documented capability has examples in 10 languages. That makes it practical to validate the DNS contract before promoting notebook code into the mail-control-plane job.

There is a real trade-off: Infrai is not a service registry, so it is the wrong choice for deploy-changing target membership. Choose Consul, Cloud Map, or Kubernetes Services for that half of the design, according to the environment that already owns the workload. Schema discoverability still does not supply deliverability evidence; the team must evaluate the MX outcome separately.

Notice what is absent from the table: a price winner. Billing does not repair stale resolution, and volatile unit prices are weak evidence for a naming decision.

## 4. Encode the boundary as an eval, not a convention

Architecture prose drifts. A focused API read makes the stable side of the rule reviewable beside deployment configuration. This Python example lists DNS records through the verified route, reads the key from the environment, sets the HTTP method explicitly, surfaces the response body on errors, and backs off on rate limiting. It does not pretend that listing DNS records proves mail delivery; the returned state is input to the MX and deliverability checks described above.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(error: HTTPError, attempt: int) -> float:
    value = error.headers.get("Retry-After")
    if value is None:
        return float(2**attempt)
    if value.isdigit():
        return float(value)
    return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())


def list_dns_records() -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    url = f"{base_url}/dns/record/list"
    for attempt in range(4):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"DNS record list failed ({error.code}): {body}"
                ) from error
            time.sleep(retry_delay(error, attempt))
    raise RuntimeError("DNS record list exhausted its retry budget")


print(json.dumps(list_dns_records(), indent=2))
```

Keep the follow-up eval small. Parse the response according to the discovered response schema, assert the expected MX state for the stable support domain, and record that result alongside independent deliverability evidence. Separately, exercise the registry during a release and rollback, watching for selection of a retired internal target. Combining those assertions into one “DNS passed” flag would hide the most important distinction in this design: control-plane state, observed mail delivery, and registry freshness answer three different questions.

I would reject a rollout gate that substitutes any one of them for the other two, even if it makes the dashboard tidier. Prompt and token cost are irrelevant here; a deterministic assertion is clearer and cheaper to operate than asking a model to interpret record state.

## 5. Retire targets, not versioned hostnames

Embedding a release version in every hostname looks explicit, but it creates a second problem: retirement work. Old names, records, and references accumulate, and the premise here is that this cleanup will not reliably happen. A registry gives changing targets an appropriate lifecycle without making the stable name carry release history.

Running DNS and a registry together is fine. Make the ownership line visible in code review: mail administrators own the durable domain and MX changes; the deployment system owns registry membership. Before copying this choice, measure whether clients ever select a retired target during deploy and rollback, and verify that the evidence used for mail deliverability remains attached to the stable public domain.

The final rule is compact: DNS names describe durable identities; registry entries describe current deployment targets. Any exception needs evidence stronger than a low TTL.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
