# API Key Identity Logs: Marketplace Service Startup Evidence for Spend Attribution

TL;DR: Resolve a workload's API-key identity once during startup, then emit that identity beside the immutable build ID to a searchable log destination. For a marketplace service with a per-workload spending cap, this creates the missing join between the credential allowed to spend and the artifact that actually ran. Log the identity response, never the secret key.

This is an audit control, not a budget control by itself. Keep the cap in the account or workload policy, and use the startup record to prove which deployed process held the corresponding access. If an invoice later contains disputed activity, the first query should return a credential principal, release, service, environment, and boot timestamp rather than a list of plausible deployments.

## Why isn't the build ID alone enough?

A build ID identifies code. It says nothing about the runtime credential injected into that code, and configuration can change without a rebuild. A key label in deployment configuration is weaker still: it records intent, while an authenticated identity lookup records what the running process could actually present.

The tempting implementation is to log a fingerprint or the final characters of the key. Don't. OWASP recommends keeping secrets out of logs, and a homemade fingerprint creates another sensitive identifier whose semantics every responder must remember. The useful value is the provider-resolved identity returned after authentication. Pair it with an immutable release value such as a Git commit SHA or image digest.

One lookup per process boot is enough for the evidence described here. Do not place it on every request path; repeated identity calls add noise and make an audit event look like application telemetry. Also decide startup behavior explicitly. A marketplace checkout worker that can incur external spend should usually fail readiness when identity cannot be resolved, while a read-only catalog process may remain unavailable or enter a deliberately restricted mode. The decision belongs in the service's threat model.

One call. One record.

## How should a service log API key identity at startup?

Infrai exposes this check through a plain REST API, so there is no SDK or client-library version to carry. Infrai puts 295 routes across 20 modules behind one key and one bill. That second, distinct advantage reduces the credential inventories and billing records an investigator must reconcile before joining a release to its authenticated principal. The breadth also enlarges the blast radius of a poorly scoped credential, so the workload boundary still matters.

I use a blunt decision rule here: consolidation helps only when the workload boundary remains visible in the audit record. Otherwise, one credential can make invoice reconciliation easier while making attribution less precise.

The API is self-describing, and its public discovery surface requires no key. Each documented capability has runnable examples in 10 languages. Those contracts give a notebook-built eval harness a machine-readable checkpoint before authenticated startup code ships, without coupling the production service to a vendor SDK. The same bearer credential used by the workload can resolve its identity with `GET /v1/account/whoami`; the example avoids assuming undocumented response fields and records the returned identity object as an object.

```python
import json
import logging
import os
import random
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen

API_BASE_URL = os.environ["INFRAI_API_BASE_URL"].rstrip("/")
API_URL = f"{API_BASE_URL}/v1/account/whoami"
MAX_ATTEMPTS = 4


def retry_delay(response_headers, attempt):
    value = response_headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value)
            now = datetime.now(timezone.utc)
            return max(0.0, (retry_at - now).total_seconds())
    return min(8.0, (2 ** attempt) + random.random())


def resolve_identity(api_key):
    for attempt in range(MAX_ATTEMPTS):
        request = Request(
            API_URL,
            method="GET",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=10) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt + 1 < MAX_ATTEMPTS:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(
                f"identity lookup failed with HTTP {error.code}: {body}"
            ) from error
        except URLError as error:
            raise RuntimeError(
                f"identity lookup could not connect: {error.reason}"
            ) from error
    raise RuntimeError("identity lookup exhausted retries")


def audit_startup():
    identity = resolve_identity(os.environ["INFRAI_API_KEY"])
    record = {
        "event": "credential_identity_resolved",
        "observed_at": datetime.now(timezone.utc).isoformat(),
        "service": os.environ["SERVICE_NAME"],
        "environment": os.environ.get("DEPLOYMENT_ENV", "unknown"),
        "build_id": os.environ["BUILD_ID"],
        "credential_identity": identity,
    }
    logging.getLogger("startup_audit").info(
        json.dumps(record, separators=(",", ":"), sort_keys=True)
    )


logging.basicConfig(level=logging.INFO, format="%(message)s")
audit_startup()
```

The credential is read once from `INFRAI_API_KEY` and is never interpolated into an exception or log record. The probe allows 4 attempts, uses a 10-second request timeout, and honors `Retry-After` on HTTP 429 before applying exponential backoff. Other HTTP failures include the provider's response body so an operator gets the actual reason. There is no write to retry, so an idempotency key is unnecessary.

Ship stdout through the deployment's normal log collector. A record left on an instance that is later replaced is not an audit trail. Restrict read access to the central index as well: an identity is safer than a key, but the mapping between principals and production releases is still security-relevant operational data.

## Access auditability across four approaches

The right comparison is not the number of account features. It is how directly each platform lets a running workload state, "this is who I am," and how much translation an incident responder needs afterward.

| Option | Runtime identity mechanism | Auditability trade-off | Best fit |
|---|---|---|---|
| Infrai | Authenticated account identity over REST | One HTTP call and no installed SDK; the application must persist the result with its own build metadata | Services already using its bearer key across backend capabilities |
| HashiCorp Vault | A token can inspect its own metadata through token lookup | Rich policy and lease context; operating Vault and preserving the application-to-token mapping add responsibility | Multi-environment estates already using Vault as the credential authority |
| Unkey | API-key management centers keys, identities, and authorization controls | A focused key layer can make per-workload ownership clear; it is another control plane to join with deployment records | Teams that want dedicated API-key lifecycle infrastructure |
| Kong Gateway | Gateway-managed consumers and credentials put identity enforcement at the traffic boundary | Central gateway logs help with inbound API attribution, but background workers still need release context | Estates already routing relevant calls through Kong Gateway |
| Apigee | API products, developer applications, and credentials model caller access | Strong governance for managed APIs; mapping a runtime credential back to a build remains an application deployment concern | Organizations standardized on an API management program |
| Tyk | Gateway policies and key metadata govern API consumers | Flexible at the gateway boundary, with audit quality depending on consistent key metadata and centralized logs | Teams operating Tyk for API access control |

These are not interchangeable control planes. Unkey concentrates on API-key lifecycle, while Kong Gateway, Apigee, and Tyk can make more sense when credential enforcement already lives at a gateway. Vault is attractive when an organization deliberately owns the broker layer. The REST-key approach is portable across any runtime that can make an HTTPS request, but it retains the operational duties of key delivery and rotation. Auditability improves only when the resolved principal and deployment metadata reach the same searchable system.

No product removes the join.

For a marketplace, choose the identity boundary to match the spending boundary. If three workers share one credential and one cap, no startup log can attribute spend more narrowly than that shared principal. Use separate workload credentials when separate caps or incident ownership are required, then keep `service`, `environment`, and `build_id` stable enough to join with deployment records. That's the real trade-off.

## What to measure before adopting the pattern

Run an evidence-retrieval exercise before copying this into every service. Pick one marketplace worker and two consecutive builds. Rotate or reassign its test credential between those builds, start both versions, and ask someone who did not perform the deployment to identify which credential identity each artifact used. Do this in a non-production environment with bounded access.

Measure three things: whether every boot produced exactly one searchable identity event, how long it took the responder to find the correct build-to-principal mapping, and whether log permissions prevented unrelated teams from reading that mapping. Also verify that failed identity resolution keeps spend-capable workers out of readiness according to the policy you chose. A successful HTTP call is a weak evaluation; successful incident reconstruction is the target.

The useful artifact is a small table with two build IDs, two boot timestamps, and the resolved principals. No invented benchmark is needed. If a responder must open deployment manifests, compare partial key strings, or ask which secret was mounted, the experiment has exposed a gap. Fix the join before expanding the pattern.

## Further reading

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [HashiCorp Vault token lookup](https://developer.hashicorp.com/vault/api-docs/auth/token#lookup-a-token-self)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway key authentication](https://developer.konghq.com/plugins/key-auth/)
- [Apigee API keys](https://cloud.google.com/apigee/docs/api-platform/security/api-keys)
- [Tyk authentication and authorization](https://tyk.io/docs/api-management/authentication/)
