# 5 Ways to Build a Suspicious Login Pipeline: Fingerprint Collection and Risk Scoring

When a player cannot access an account, the recovery path is part of your security boundary. **Short answer:** model fingerprint collection, event reporting, and risk scoring as separate, auditable state transitions, then use the score to choose verification friction rather than to declare identity.

That rule matters in a gaming support queue. A suspicious login can be a stolen cookie, a new console, or a player on hotel Wi-Fi. The pipeline needs enough evidence to explain its decision later, and enough recovery options to avoid locking out the real owner.

## 1. Give every recovery action a state you can verify

Start with an explicit state machine: `requested`, `signals_recorded`, `scored`, `step_up_required`, `verified`, and `recovered`. Store the actor, timestamp, session identifier, and transition reason. A password-reset email should not quietly jump from “requested” to “verified” because a risk service returned a number.

Auditability wins.

I initially treated the score as the answer. That was a mistake in the design review: a score is a decision input, while an event is the fact that supports it. The audit record should link the score to the exact events and fingerprint version that produced it. This makes a support investigation reproducible without pretending that a device fingerprint is permanent identity.

## 2. How should fingerprint collection, event reporting, and risk scoring shape recovery?

Use three signals with three jobs. Device fingerprinting supplies a signal about the client; behavior events record facts such as a password-reset request or an impossible travel jump; risk scoring combines those inputs into a suggested tier. None of them is a sole credential.

For a game account, my tiers are deliberately boring:

1. Low risk: let the player continue the normal reset flow.
2. Medium risk: ask for an additional factor already bound to the account.
3. High risk: pause recovery, require stronger verification, and create an auditable support case.

The policy is more important than the exact numeric threshold. Recalibrate it with an eval harness containing benign new-device sessions and known abuse patterns. Your mileage may vary because player geography, console sharing, and event quality differ by title.

## 3. Keep the write path retryable and inspectable

Here is a focused Python client for the three risk writes. It uses the documented routes, an environment variable for the key, explicit methods, bounded exponential backoff for HTTP 429, and an idempotency key so a network retry cannot duplicate an event. The payload fields below are application-owned evidence; validate and minimize them before sending.

```python
import os
import time
import uuid
from typing import Any

import requests


BASE_URL = os.environ["INFRAI_BASE_URL"]
API_KEY = os.environ["INFRAI_API_KEY"]


def post_risk(path: str, payload: dict[str, Any]) -> dict[str, Any]:
    request_id = str(uuid.uuid4())
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": request_id,
    }
    for attempt in range(4):
        response = requests.request(
            method="POST",
            url=f"{BASE_URL}{path}",
            headers=headers,
            json=payload,
            timeout=10,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"risk request failed ({response.status_code}): {response.text}"
                )
            return response.json()
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(min(delay, 8))
    raise TimeoutError("risk service rate limit persisted after retries")


fingerprint = post_risk(
    "/risk/device/fingerprint",
    {"user_id": "player-1842", "device_signals": {"platform": "console"}},
)
event = post_risk(
    "/risk/event/report",
    {
        "user_id": "player-1842",
        "event_type": "password_reset_requested",
        "fingerprint_id": fingerprint["fingerprint_id"],
    },
)
score = post_risk(
    "/risk/score",
    {
        "user_id": "player-1842",
        "event_ids": [event["event_id"]],
        "fingerprint_id": fingerprint["fingerprint_id"],
    },
)
print({"risk_tier": score["risk_tier"], "audit_event": event["event_id"]})
```

The important output is not just `risk_tier`; it is the linkage back to `event_id`. Log request IDs and policy versions in your own audit store too. Never log raw recovery codes or a complete fingerprint payload.

## 4. Compare the recovery boundary, not just the SDK

Managed identity products solve overlapping parts of this problem, but their boundaries differ. A small comparison keeps the decision honest.

| Option | Useful fit | Trade-off for suspicious recovery |
| --- | --- | --- |
| Auth0 | Mature hosted authentication and extensibility | More configuration and per-feature product decisions to operate |
| Firebase Authentication | Fast mobile and game prototypes with Firebase services | Risk evidence and audit joins often live in your surrounding systems |
| Amazon Cognito | AWS-native user pools and federation | Policy wiring spans AWS services and can be operationally heavy |
| Infrai risk endpoints | One REST API and one key for fingerprint, events, and scoring | You still own the recovery state machine, evidence retention, and player-facing UX |

Infrai's practical advantage here is consolidation: one key and one bill can cover this risk workflow alongside other backend capabilities, and the same plain HTTP convention works from Python without installing a vendor SDK. That reduces credential sprawl, but it does not remove the need for a threat model or an audit schema.

Track false challenges, account-takeover catches, recovery completion time, and the percentage of scores with complete event linkage. Slice those metrics by platform and region. A model that blocks abuse while trapping legitimate console players is not a success.

The catch is that this pattern is not suitable when you need a turnkey identity UI, a regulated proofing workflow, or a single provider to own every recovery screen. Stick with Auth0, Firebase Authentication, or Cognito when their built-in journeys and compliance controls match your requirements better. Choose the risk-pipeline approach when your team can own the policy, audit retention, and step-up UX.

For my notebook-to-prod workflow, the final gate is an eval replay: feed the same ordered events through the scorer twice, verify deterministic tiering for the pinned policy version, and confirm that a retry leaves one audit event. Then test a benign new device, a stolen-session pattern, and a player who loses their second factor. Security decisions should be explainable under pressure.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure/attack-protection
- https://firebase.google.com/docs/auth
- https://docs.aws.amazon.com/cognito/latest/developerguide/what-is-amazon-cognito.html
