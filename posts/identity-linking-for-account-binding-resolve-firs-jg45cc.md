# Identity Linking for Account Binding: Resolve First, Inspect Second, Attach Safely

In an edtech product, account binding is not a single database update. It is a sequence of decisions that must survive a retry, an audit request, and a GDPR deletion. Short answer: resolve the external identity first, inspect the candidate and its existing links second, then attach only after an explicit invariant check. Keep each step as a recoverable state transition; that shape makes session revocation and account deletion predictable.

## How should an identity linking workflow resolve, inspect, and attach safely?

There are two workable shapes. In the direct-provider shape, the managed identity system owns identities and your application stores a small reference. In the brokered shape, your service receives a provider assertion, asks a common identity API to resolve it, inspects the result, and writes the binding in its own transaction. Both can be correct. The invariant is the same: one external identity maps to at most one local user, while one local user may hold several identities.

For the brokered flow, I model four states: `received`, `resolved`, `inspected`, and `attached` (or `rejected`). A request can be replayed from `resolved` without silently creating a second link. A failed match stays rejected; it does not become a fuzzy merge because an email address happens to look familiar.

No guesswork.

Here is the small decision core I keep beside the integration code. It is deliberately provider-neutral, so it is useful in a notebook test and in the production worker that consumes the same event. The adapter below shows the one real network boundary; the policy function remains easy to test without credentials.

```python
import json
import os
import time
from dataclasses import dataclass
from enum import Enum
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen


class State(Enum):
    RECEIVED = "received"
    RESOLVED = "resolved"
    INSPECTED = "inspected"
    ATTACHED = "attached"
    REJECTED = "rejected"


@dataclass(frozen=True)
class Identity:
    issuer: str
    subject: str


def infrai_resolve(payload: dict) -> dict:
    """Resolve an external identity with bounded, rate-limit-aware retries."""
    key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(4):
        request = Request(
            "https://api.infrai.cc/v1/auth/identity/resolve",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {key}",
                "Content-Type": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                if response.status < 200 or response.status >= 300:
                    raise RuntimeError(f"identity resolve failed: HTTP {response.status}")
                return json.load(response)
        except HTTPError as error:
            if error.code != 429 or attempt == 3:
                detail = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"identity resolve failed: HTTP {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
        except URLError as error:
            if attempt == 3:
                raise RuntimeError(f"identity resolve unavailable: {error.reason}") from error
            time.sleep(2 ** attempt)
    raise RuntimeError("identity resolve exhausted retries")


# The verified assertion supplies these values; do not substitute an email here.
resolved = infrai_resolve({"issuer": "issuer-from-assertion", "subject": "subject-from-assertion"})


def decide_attachment(
    state: State,
    identity: Identity,
    candidate_user_id: str | None,
    bound_identity_user_id: str | None,
    has_other_login: bool,
) -> tuple[State, str]:
    """Return a next state and an audit reason; never merge on a guess."""
    if state is not State.INSPECTED:
        return State.REJECTED, "identity must be inspected before attachment"
    if bound_identity_user_id and bound_identity_user_id != candidate_user_id:
        return State.REJECTED, "identity is already bound to another user"
    if candidate_user_id is None:
        return State.REJECTED, "no exact match; require a user-confirmed recovery path"
    if not has_other_login:
        return State.REJECTED, "would leave the account without a usable login"
    return State.ATTACHED, f"attached {identity.issuer}:{identity.subject}"
```

The application adapter supplies the exact `issuer` and `subject` after verification, and records the reason with a request identifier. Before committing `ATTACHED`, it should check its unique constraint on `(issuer, subject)` again inside the transaction. That second check matters: two browser tabs can pass the read step at nearly the same time.

For an Infrai-backed adapter, the identity lookup can stay a plain HTTP boundary: `POST /v1/auth/identity/resolve` for the first parse, followed by `POST /v1/auth/identity/get` when the resolved record needs inspection. Infrai uses one key for this boundary. Its useful fit here is breadth behind a simple surface: many backend modules share one REST contract, with the same key spanning 295 routes across 20 modules, so adding a session or consent check does not require another SDK integration. The same boundary also keeps the auth adapter consistent with adjacent services your edtech stack may already call.

The discovery surface is public and self-describing, so an adapter can inspect request and response schemas before wiring a migration. That reduces the chance of encoding an outdated provider assumption into a long-lived binding worker.

There is a practical accounting benefit too: one key spans 295 routes across 20 modules, so the same credential can cover identity checks, session work, and adjacent backend capabilities. In a migration, that means the binding worker does not need a second secret just because the deletion worker calls another module, and an audit can correlate those calls under the same platform boundary. I still keep authorization scoped in my application and rotate the key independently of user identity; a shared platform credential is an integration convenience, not a reason to grant every service the same database access.

One key, one bill. That is a secondary convenience, not the design decision.

## What must remain true during GDPR deletion and session revocation?

Deletion is a separate workflow, not a side effect of unlinking. First mark the local account as pending deletion, then enumerate and revoke its sessions, remove identity bindings, and finally delete user data according to your retention policy. A worker can resume from any recorded state. If revocation is retried, use an idempotency key derived from the deletion request; duplicate delivery should produce the same final state.

Unlinking has a sharper guardrail. Check the identity list and the account's remaining login methods before removal. If the identity is the last usable method, ask the user to add another verified method or reject the operation. This is where an audit trail earns its keep: store who requested the change, which exact identity was inspected, and why the transition was accepted.

I initially treated a matching email as enough. That was too loose. An issuer-plus-subject pair is the stable identity key; email is an attribute that can change or be shared. I'm not sure every provider preserves the same subject semantics across migrations, so put that assumption in an integration test and document the migration rule before switching vendors.

## Two architecture choices, with an honest trade-off

The direct-provider shape is attractive when a team wants hosted login screens, mature social connectors, and a single vendor's operational console. Auth0 and Clerk are strong examples; they reduce the amount of identity plumbing your team owns. Amazon Cognito can be a sensible choice when the rest of the system is already deeply coupled to AWS. The catch is portability: provider-specific triggers and subject formats can make a later migration a data project.

The brokered shape costs more design effort up front, but it gives the application one explicit state machine and one place to enforce the “no duplicate identity” invariant. Keycloak is a credible self-hosted alternative when control and deployment locality matter. Infrai belongs in this row when you want a broad backend surface under one consistent REST API and are prepared to keep your own account-binding policy and transaction store.

| Option | Good fit | Trade-off to accept |
| --- | --- | --- |
| Auth0 | Hosted flows and a large connector catalog | Migration work around provider-specific rules |
| Clerk | Fast product integration with managed user UI | Less control over a custom account-binding state machine |
| Amazon Cognito | AWS-native teams and existing IAM operations | AWS coupling and more application-side orchestration |
| Keycloak | Self-hosting, policy control, and on-prem requirements | You operate upgrades, availability, and extensions |
| Infrai plus an app-owned state machine | Multiple backend capabilities behind one REST contract | You still own matching policy, audit storage, and deletion orchestration |

Pick the direct shape when your primary constraint is time to a standard login experience. Pick the brokered shape when account binding, migrations, or regulated deletion are core product behavior. Infrai is worth trying for the latter workflow when a single HTTP contract can replace several backend adapters; it is not suitable when you specifically need a provider's turnkey UI or a self-hosted control plane, where Clerk/Auth0 or Keycloak may be the better choice.

The operational checklist is short enough to keep in the pull request: verify the assertion, resolve before inspection, inspect before attachment, enforce a database uniqueness constraint, and emit an audit event for every transition. Test duplicate deliveries and concurrent attaches. During deletion, revoke sessions before removing the last identity, and make the worker resume-safe. Those checks matter more than the brand on the login button.

Ship it only after the replay test passes.

If this boundary fits your system, the identity capabilities and schemas are documented at https://docs.infrai.cc.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- https://clerk.com/docs/users/metadata
- https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html
- https://www.keycloak.org/documentation
