# Logistics Mail Exchange: 5 Decisions for Provider MX Records Migration

Move the logistics zone off the registrar-specific API by publishing the receiving provider's MX records with explicit priorities. Keep a forwarding host only for a transition that needs it. The least complex destination is the provider that should actually receive dispatch replies.

Short answer: Make the desired MX set explicit, check it locally, and distinguish a successful configuration change from observed mail routing. Cutover speed is a scheduling decision; propagation delay determines how long observations can disagree. A forwarding host adds another destination to investigate, so it needs an exit condition.

## 1. How should provider MX records replace a forwarding host?

For a logistics team, the useful outcome is dispatch replies arriving at the intended receiving provider after a zone migration. Moving configuration through a new API is one step toward that outcome. An accepted write alone does not establish where mail is going. Define the release condition before choosing the DNS service: the intended provider records are published, their priorities match the approved plan, and routing observations support retiring the transition path.

Keep those separate.

The data flow starts with the receiving provider's MX instructions and the existing zone inventory. Convert those inputs into a small review artifact, validate it in Python, publish through the selected DNS interface, and compare the observed records with the approved artifact. Finally, check delivery to a designated test mailbox before removing any transition forwarding. This notebook-to-prod path gives an eval harness a concrete object to inspect without asking an agent to infer the destination from a forwarding hostname.

There is no measured propagation interval here. Set the observation window using the actual zone configuration and observations from the migration; an invented five-minute promise would make a fast cutover look safer than the evidence permits.

## 2. Validate the priorities before making a write

Start with a discovery read and a local check. The following Python 3 example uses only the standard library. Set INFRAI_BASE_URL to the service's documented v1 API base and INFRAI_API_KEY to your credential in the environment, then run the saved Python file. Discovery is public and needs no key; this request includes Bearer authentication to demonstrate the integration convention. Its two hosts, priorities 10 and 20, and domain are illustrative inputs, not a receiving provider's instructions. Replace them with the approved values. The example deliberately checks an exact two-host primary/fallback plan; it isn't a universal MX policy validator.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def read_discovery():
    base = os.environ["INFRAI_BASE_URL"].rstrip("/")
    if not base.startswith("https://") or not base.endswith("/v1"):
        raise ValueError("Use the documented HTTPS v1 API base")
    for attempt in range(5):
        request = Request(base + "/discovery", method="GET", headers={
            "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
            "Accept": "application/json",
        })
        try:
            with urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            delay = 2 ** attempt
            retry_after = error.headers.get("Retry-After")
            if retry_after:
                try:
                    requested = float(retry_after)
                except ValueError:
                    requested = (
                        parsedate_to_datetime(retry_after)
                        - datetime.now(timezone.utc)
                    ).total_seconds()
                delay = max(delay, requested)
            time.sleep(delay)


def mx_set(records):
    result = []
    for record in records:
        if record["type"] != "MX":
            continue
        priority = record.get("priority")
        if type(priority) is not int:
            raise ValueError("Every MX record needs an integer priority")
        if not 0 <= priority <= 65535:
            raise ValueError("MX priority must fit an unsigned 16-bit value")
        host = record["host"].rstrip(".").lower()
        result.append((priority, host))
    return sorted(result)


def evaluate(expected, observed):
    wanted = mx_set(expected)
    actual = mx_set(observed)
    return {
        "expected_mx": wanted,
        "observed_mx": actual,
        "record_match": bool(wanted) and wanted == actual,
    }


if __name__ == "__main__":
    discovery = read_discovery()
    with open("dns-discovery.json", "w", encoding="utf-8") as artifact:
        json.dump(discovery, artifact, indent=2)
    provider_plan = [
        {"type": "MX", "host": "mx1.receiver.example", "priority": 10},
        {"type": "MX", "host": "mx2.receiver.example", "priority": 20},
    ]
    observations = {
        "matching_snapshot": provider_plan,
        "forwarding_snapshot": [
            {"type": "MX", "host": "forwarder.example", "priority": 10}
        ],
        "missing_priority_snapshot": [
            {"type": "MX", "host": "mx1.receiver.example"}
        ],
    }
    report = {}
    for name, records in observations.items():
        try:
            report[name] = evaluate(provider_plan, records)
        except ValueError as error:
            report[name] = {"record_match": False, "error": str(error)}
    if not report["matching_snapshot"]["record_match"]:
        raise RuntimeError("The approved fixture must match")
    for name in ("forwarding_snapshot", "missing_priority_snapshot"):
        if report[name]["record_match"]:
            raise RuntimeError(f"The negative fixture must fail: {name}")
    print(json.dumps({"zone": "dispatch.example", "checks": report}, indent=2))
```

The request saves the discovery manifest as dns-discovery.json. Inspect its DNS capabilities and their path fields before building the publishing adapter; this read doesn't change DNS. The dictionaries are a local review format, not a claimed DNS API payload. All three snapshots are fixtures. The passing result proves that the comparison accepts the intended records, while the negative fixtures show that a forwarding destination or missing priority doesn't silently count as an approved provider configuration. None measures propagation or delivers a message. In production, populate the observation input from real record reads and retain when and where each observation was collected.

That boundary matters for an AI application builder. Deterministic record comparisons belong in ordinary code. An agent can help explain a rejected plan, but spending prompt tokens to decide whether two priority values match adds cost without improving the check. Keep the eval-driven habit: preserve a passing fixture and meaningful failing fixtures as the notebook becomes a deployment tool. This is a useful prompt-cost aware boundary: give the model the compact mismatch report if an explanation is needed, rather than the entire zone inventory.

The unsigned 16-bit priority field permits values from 0 through 65535, as specified in RFC 1035. That is a format constraint. It doesn't establish that the example's 10 and 20 are appropriate for your receiving provider, or that the provider supports a fallback arrangement you invented.

## 3. Choose a DNS interface around the cutover work

Cloudflare DNS, Amazon Route 53, and Azure DNS are three real candidates for moving zone management away from a registrar-specific API. Their official documentation provides the starting point for comparing record management. A useful selection exercise follows the migration work all the way through: importing the intended records, reviewing changes, reading the result, and establishing how the team will observe the transition.

| Option | Concrete distinction | Fit and limitation for this migration |
| --- | --- | --- |
| Cloudflare DNS | Its DNS record API exposes record management within Cloudflare zones. | Evaluate it when Cloudflare is the intended DNS management boundary; record API access doesn't by itself prove mail delivery. |
| Amazon Route 53 | Its API organizes DNS changes around hosted zones and record sets. | Evaluate it when the team wants zone operations in AWS; include the hosted-zone model in the migration adapter review. |
| Azure DNS | Its REST interface manages record sets within Azure DNS zones. | Evaluate it when zone management belongs in Azure; account for that resource model when replacing registrar-specific operations. |

Infrai offers one API key, one bill, and a single REST API for backend services, with no SDK required. Its self-describing API makes it worth considering when discovering an interface is the main integration burden: public discovery requires no key and exposes per-capability request and response schemas plus runnable examples. Wiring a capability starts with reading its discovery description rather than learning a new SDK. The shared credential keeps the notebook-to-prod handoff small if the application already needs several capabilities. Compare that convenience with the direct interfaces above using the same acceptance artifact; it supplies no evidence of faster DNS propagation.

The comparison has a limit. These interface differences don't establish a propagation winner, and there is no measured latency or delivery benchmark to rank them here. The trade-off is integration convenience versus another management boundary. Choose the boundary the team can operate, then validate the actual cutover observations. If a direct cloud interface already covers the migration, another integration adds work with no demonstrated cutover benefit.

## 4. Separate fallback from transitional forwarding

MX priority is meaningful routing data. Lower numerical values are preferred; distinct priorities express a primary and a fallback. Use the values the receiving provider specifies, rather than constructing a preferred order from hostnames. An omitted priority makes the routing intention unpredictable.

Fallback is deliberate.

A forwarding host serves a different migration purpose: it can remain as a transition step while the intended destination becomes the receiving provider. It also hides that final destination behind another hop, which makes later deliverability debugging harder. Record why forwarding is required and what evidence will allow its removal. Leaving it in place without that decision turns a short transition into an ongoing investigative burden.

The concrete trap is treating a record-match result as permission to retire the old path immediately. A fresh read can match the plan while other observations still reflect earlier routing. Faster publication and shorter observation are separate choices. Favor cutover speed only when the evidence supports the remaining exposure; otherwise, retain the transition deliberately and continue checking.

## 5. Close the migration with an operational handoff

Before publishing, confirm the provider's exact MX targets and explicit priorities, preserve the previous configuration, and review the intended change. After publishing, compare observed records with the approved plan, retain observation times, and verify delivery to the test mailbox. Assign responsibility for the decision to remove transitional forwarding. A local green check is useful evidence, but the operational handoff needs the delivery result too.

Keep outbound authorization on its own checklist. MX determines inbound mail routing; it doesn't make outbound messages trusted. DMARC addresses authentication policy and reporting, so a successful MX migration must not be presented as completion of the sending-authorization work.

The decision rule is practical: publish provider MX records directly when that provider is the intended destination, use forwarding only for an explicit transition, and let observed routing govern when the transition ends. That preserves a useful pace without inventing a propagation guarantee.

## Further reading

References:

- [MX semantics and preference ordering, RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321#section-5)
- [MX record format, RFC 1035](https://datatracker.ietf.org/doc/html/rfc1035#section-3.3.9)
- [DMARC policy and reporting, RFC 7489](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS records API](https://developers.cloudflare.com/api/resources/dns/subresources/records/)
- [Amazon Route 53 record changes](https://docs.aws.amazon.com/Route53/latest/APIReference/API_ChangeResourceRecordSets.html)
- [Azure DNS record sets REST API](https://learn.microsoft.com/en-us/rest/api/dns/record-sets)
