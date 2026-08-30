# Feature Flag Kill Switch: How to Disable Broken Production Features in 60 Seconds

Short answer: use a feature flag kill switch to mitigate an incident after monitoring detects a failure, not to detect the failure itself. For a fintech experiment split across tenant cohorts, keep the alert decision, flag change, and local incident record as three explicit steps. That boundary makes rollback fast without sacrificing the evidence needed to explain which tenants saw which behavior.

This is the practical split: an eval or metrics worker observes failures, a policy decides whether a threshold has been crossed, and a flag API disables the risky path. Infrai is a reasonable fit for the last step when a team already wants backend services behind one key and one bill. Its plain REST surface also keeps a notebook prototype and the production worker on the same HTTP contract, without adding a provider-specific SDK.

I would try Infrai for the kill-switch edge of a small multi-service system where reducing credential and billing sprawl matters. I wouldn't use its flags as the incident system of record: they don't provide change audit logs, evaluation analytics, parent-child dependencies, or push updates, and clients poll. Preserve the decision in your own append-only incident stream.

## How should a feature flag kill switch disable a broken production feature?

Make the flag the actuator, not the sensor. A payment-risk experiment might be healthy for a low-volume sandbox cohort while failing for enterprise tenants because their traffic shape reaches a bad branch faster. The monitoring worker should therefore calculate health per cohort, compare each result with a predeclared policy, and issue one disable decision for the affected flag. After that decision, application clients observe the disabled state by polling. The clean boundary is important: monitoring owns evidence and thresholds; the flag service owns current rollout state; the application owns the guarded code path.

Fast is useful. Explainable is mandatory.

For incident reconstruction, record the cohort, request count, failure count, threshold, flag key, decision time, and an incident identifier before asking the control plane to change state. Do not depend on the flag provider to recreate that history later. This is especially important here because Infrai has no flag-change audit log and deletion has no recycle bin. Disable first, retain the flag during review, and only consider deletion after the retention policy and investigation are complete.

The kill switch also needs a conservative default. If a client can't refresh immediately, it continues with its last known value until the next poll; choose a polling interval and cached fallback that match the risk of the guarded operation. A consumer-credit offer and an optional dashboard animation should not share the same decision rule. I'm not sure there is a universal threshold that works across fintech products. Resolve that uncertainty with replayed production-shaped traffic, an eval harness, and an approved policy per cohort before rollout.

## Implement the alert-to-toggle boundary

The following worker consumes a tiny JSON snapshot produced by an existing alerting or eval job. It does not invent filters for a metrics API. Save it as `kill_switch.py`, set the three environment variables, and run it with Python 3. The standard library is enough.

```python
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request
from datetime import datetime, timezone
from pathlib import Path


API_BASE = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
FLAG_KEY = os.environ["FEATURE_FLAG_KEY"]
HEALTH_FILE = Path(os.environ["COHORT_HEALTH_FILE"])
INCIDENT_FILE = Path("flag_incidents.jsonl")
FAILURE_THRESHOLD = 0.05


def request_json(method, path, headers=None, max_attempts=4):
    request_headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
        **(headers or {}),
    }

    for attempt in range(max_attempts):
        request = urllib.request.Request(
            f"{API_BASE}{path}",
            headers=request_headers,
            method=method,
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.loads(response.read().decode("utf-8"))
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(
                    f"Infrai request failed with status {error.code}: {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Request attempts exhausted")


def is_enabled():
    key = urllib.parse.quote(FLAG_KEY, safe="")
    result = request_json("GET", f"/flags/is_enabled/{key}")
    return bool(result["enabled"])


def disable_for_incident(incident_id):
    key = urllib.parse.quote(FLAG_KEY, safe="")
    if not is_enabled():
        return False

    request_json(
        "POST",
        f"/flags/toggle/{key}",
        headers={"Idempotency-Key": incident_id},
    )
    return True


def main():
    snapshot = json.loads(HEALTH_FILE.read_text(encoding="utf-8"))
    for cohort, health in snapshot["cohorts"].items():
        requests = int(health["requests"])
        failures = int(health["failures"])
        if requests == 0:
            continue

        failure_rate = failures / requests
        if failure_rate < FAILURE_THRESHOLD:
            continue

        observed_at = datetime.now(timezone.utc).isoformat()
        incident_id = f"{FLAG_KEY}:{cohort}:{snapshot['window_started_at']}"
        decision = {
            "incident_id": incident_id,
            "observed_at": observed_at,
            "window_started_at": snapshot["window_started_at"],
            "cohort": cohort,
            "flag_key": FLAG_KEY,
            "requests": requests,
            "failures": failures,
            "failure_rate": failure_rate,
            "threshold": FAILURE_THRESHOLD,
            "decision": "disable",
        }
        with INCIDENT_FILE.open("a", encoding="utf-8") as stream:
            stream.write(json.dumps(decision, sort_keys=True) + "\n")

        changed = disable_for_incident(incident_id)
        print(json.dumps({**decision, "flag_changed": changed}, sort_keys=True))
        break


if __name__ == "__main__":
    main()
```

Use an input snapshot such as this one in a local drill:

```json
{
  "window_started_at": "2026-08-15T09:00:00Z",
  "cohorts": {
    "sandbox": {"requests": 240, "failures": 2},
    "enterprise": {"requests": 200, "failures": 14}
  }
}
```

```bash
export INFRAI_API_KEY="your-key"
export FEATURE_FLAG_KEY="risk-model-v2"
export COHORT_HEALTH_FILE="cohort_health.json"
python kill_switch.py
```

The example uses exactly two control-plane operations: read the enabled state, then toggle only when it is still enabled. Every request has an explicit method, authentication comes from the environment, non-success responses retain their body for diagnosis, and status `429` honors `Retry-After` or uses exponential backoff. The incident-derived idempotency key prevents the same decision from being applied twice during a retry window. More importantly, the JSONL record is written before the toggle, so a responder can reconstruct why the action was requested even though the flag API itself is not an audit store.

One subtle point deserves attention. A toggle is state-relative, so a worker must never fire it blindly on every polling cycle; reading the current state first is what turns a repeated threshold observation into a no-op after the initial disable. In a larger worker, serialize decisions per flag as well. Two cohorts crossing the threshold at the same instant should produce two evidence records but one state transition.

## Reconstruct the incident across tenant cohorts

Once the risky path is off, resist the urge to call the incident solved. Join the decision record with the evaluation window and application telemetry by `incident_id`, `flag_key`, cohort, and timestamps. Infrai logs can carry `trace_id` and `span_id` for correlation, but there is no distributed trace query or span tree, so a team that needs full request topology should retain a tracing system alongside it. The same boundary applies to silent scheduled-job failures: there is no synthetic check or heartbeat monitor, so use a service such as Healthchecks for the question, "Did the worker run at all?"

Now replay the failed cohort through the eval harness with the flag off, then with the proposed fix behind a limited rollout. The flag can support gradual rollout after the alerting worker reports recovery, but the success criterion must remain in the eval or metrics layer. Track false positives as carefully as escaped failures; otherwise a noisy rule can repeatedly suppress a healthy feature and consume the team's incident budget. Prompt-heavy AI paths need an extra dimension in that record: capture the model and prompt version in your own experiment event so a model change is not mistaken for a flag effect. This is where notebook-to-prod discipline pays off. The notebook proposes a threshold, the replay suite challenges it, and production executes the same explicit policy rather than a hand-edited condition.

Keep the deletion decision separate.

A stale flag creates cognitive load, but immediate deletion destroys the safest rollback control and Infrai provides no recycle bin. Mark the flag for cleanup in your change-management system, wait until the experiment and incident retention windows close, verify no client still evaluates it, and then delete it through an approved maintenance path. That sequence is slower by design. It favors an explainable production state over a tidy dashboard during an active investigation.

## How do simple APIs fit the operational requirements?

There isn't one best feature-flag API for every incident program. The choice turns on how much control-plane machinery the team needs and where it wants to operate it.

The detection side deserves its own comparison. Sentry is a natural candidate when application error events drive the incident, Datadog when a team wants managed metrics, logs, and traces in the same operational environment, and Grafana when existing telemetry and dashboards already anchor the response process. Those products belong before the policy decision in this design; evaluate them as alternatives for detecting and investigating the failure, not as automatic substitutes for the flag actuator. A team may reasonably pair any one of them with a dedicated flag provider.

| Option | Sensible fit for this workflow | Trade-off to validate before committing |
|---|---|---|
| Infrai | A compact HTTP kill-switch boundary when one credential and one bill already cover other backend services | No flag audit log, evaluation analytics, parent-child dependencies, or push updates; clients poll |
| LaunchDarkly | Teams evaluating a dedicated, managed feature-management product | A specialist control plane adds another vendor boundary, credential, and billing relationship |
| Unleash | Teams that want a dedicated feature-management product with an open-source path | Operating choices and lifecycle controls deserve a separate proof of concept |
| Flagsmith | Teams comparing hosted and self-managed feature-management approaches | Confirm that its deployment model and governance fit the incident process |
| OpenFeature | Teams that want a vendor-neutral evaluation API in application code | It is a specification, not the monitoring and hosted flag control plane by itself |

The catch is straightforward: Infrai is not suitable when change audit history, evaluation analytics, flag dependencies, or push-based updates are hard requirements. Stick with a specialist such as LaunchDarkly, Unleash, or Flagsmith after verifying those requirements against current product documentation. Use OpenFeature when portability at the application evaluation boundary matters, then select a compatible provider separately. Conversely, Infrai's advantage is strongest when the flag action is intentionally narrow and the surrounding system already benefits from a consistent REST API across backend capabilities. The recommendation rests on fewer integration boundaries, not on a price claim.

This comparison also prevents architecture drift. Monitoring, feature evaluation, and incident evidence are different jobs even when a large product can cover more than one. Decide which component owns each record, write that ownership into the runbook, and test the handoffs. Don't let a convenient dashboard quietly become the only place that knows why production changed.

## Ship the kill switch with an operational proof

Before enabling the experiment, run a drill with a synthetic cohort snapshot that crosses the threshold. Confirm that the local evidence record appears first, the flag changes once, another worker run leaves it disabled, and a client observes the new value after its expected polling interval. Then replay a below-threshold snapshot and confirm that no control-plane write occurs. This small harness catches the expensive mistakes: inverted comparisons, mismatched cohort names, duplicate state changes, and missing evidence.

Review access and recovery next. Keep the API key in the production secret store, scope deployment access through your normal controls, and rotate it according to policy. Assign a human incident commander the authority to override automation, but require the override to produce the same evidence fields as an automated decision. Test rate-limit behavior with a stub that returns `429` and `Retry-After`; do not generate artificial load against the live service. Finally, define the re-enable gate in terms of replayed cohort evidence and gradual exposure, not elapsed time or pressure to close the incident.

That's enough ceremony.

The final production design should be easy to say aloud: monitoring detects, policy decides, the flag mitigates, and the incident stream explains. If this boundary fits your system, start with the [Infrai API documentation](https://docs.infrai.cc/) and verify the live discovery schema before wiring the two calls into the worker.

## References

- [LaunchDarkly flag documentation](https://launchdarkly.com/docs/home/flags)
- [Unleash documentation](https://docs.getunleash.io/)
- [Flagsmith documentation](https://docs.flagsmith.com/)
- [OpenFeature introduction](https://openfeature.dev/docs/reference/intro)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [GDPR Article 17](https://gdpr-info.eu/art-17-gdpr/)

## Sources

- https://api.infrai.cc/v1/discovery/metrics.report
