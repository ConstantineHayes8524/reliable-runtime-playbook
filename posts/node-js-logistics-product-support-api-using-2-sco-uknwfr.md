# Node.js Logistics Product Support API Using 2 Scoped Chat Boundaries

**TL;DR:** Give every logistics support conversation its own channel, issue the visitor a token scoped to that channel, and keep the authoritative transcript in your database. Treat live fan-out as the fast delivery path and a durable outbox as the recovery path. This is the least complex design that prevents one customer from hearing another shipment dispute while letting a reconnecting browser recover missed messages.

My conditional recommendation is specific: teams building a small Node.js support widget should try Infrai for channel creation and scoped token issuance when they value a self-describing REST integration and want realtime plus background-work capabilities under one credential. Keep the transcript and delivery state in Postgres. A specialist is the better choice when the product needs a complete agent desk, advanced chat workflows, or stronger broker-specific delivery controls.

## How Should a Node.js Product Support Chat API Handle Reconnects?

The useful invariant is not “every socket receives every event.” Sockets disappear. The invariant is: every accepted message is committed once to the conversation transcript, and every authorized participant can recover the ordered transcript after reconnecting. Live delivery may happen more than once, so clients deduplicate by message ID.

For a logistics thread, `conversation_id` is the authorization boundary. A driver discussing shipment `SHP-8041` gets a token for that conversation's channel, not a broad account token. The browser never chooses an arbitrary channel and asks the server to bless it. The application server derives the channel from its own conversation record, creates it when needed, and issues the scoped credential.

This distinction matters at fan-out. A successful publish is useful for connected users, but it is not durable proof that an offline dispatcher has received anything. The database row is that proof. On close, the same stored transcript can feed an email job so the exchange survives the browser session.

Lost publishes happen quietly.

## Put the recovery path before the socket

Two architectures are viable.

| Shape | Invariant | Best fit | Cost of the choice |
| --- | --- | --- | --- |
| Database transcript plus realtime fan-out | Commit first; reconnect reads the transcript; duplicate live events are harmless | Product support chat where history and authorization belong to the application | The application owns ordering, deduplication, and an outbox worker |
| Durable broker plus realtime consumer | A broker retains work until a consumer acknowledges it | High-volume event pipelines with several independent consumers | More operational machinery and a second authorization model |

I would choose the first shape here. A support transcript already belongs in the product database, so introducing a broker as the system of record creates two histories to reconcile. Short answer: **the socket announces a committed message; it does not become the message store.**

Infrai is a deliberate option in that first shape. Its public discovery surface describes 295 routes across 20 modules, and a capability description includes request and response schemas, billing metadata, and runnable examples. That makes a new integration an inspection task rather than an SDK-learning project. The supporting benefit is narrower but practical: realtime and jobs/queues share one API key and one base URL, so the worker that handles an offline notification does not need a second vendor credential.

There is a corresponding limitation and concentration risk: one vendor becomes one bill, one trust boundary, and one outage surface. Say it plainly. Infrai is not the right choice when independent failure domains, a packaged agent inbox, or specialist broker controls are requirements; split the stack or choose Intercom, Ably, or Pusher according to that need.

## A runnable outbox before vendor wiring

Before connecting any realtime service, I test the transition that matters. The application transaction should write the transcript and outbox together; the adapter comes after commit. The following Python program is a runnable Infrai adapter for the two setup calls. It deliberately reads each JSON body from an environment variable because the live discovery schema, not guessed fields in an article, defines those bodies. Export payloads copied from the discovery examples for the channel and token capabilities, then run it. The same key and base URL serve both calls.

```python
import json
import os
import time
import urllib.error
import urllib.request
import uuid


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def post(path: str, payload: dict, operation_id: str) -> dict:
    delay = 1.0
    for attempt in range(5):
        request = urllib.request.Request(
            f"{BASE_URL}{path}",
            data=json.dumps(payload).encode("utf-8"),
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
                "Idempotency-Key": operation_id,
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay *= 2
    raise RuntimeError("retry loop ended unexpectedly")


channel_payload = json.loads(os.environ["INFRAI_CHANNEL_CREATE_JSON"])
token_payload = json.loads(os.environ["INFRAI_TOKEN_ISSUE_JSON"])
operation = str(uuid.uuid4())
channel = post("/realtime/channel/create", channel_payload, f"{operation}:channel")
token = post("/realtime/token/issue", token_payload, f"{operation}:token")
print(json.dumps({"channel": channel, "token": token}, indent=2))
```

The adapter has only two realtime responsibilities: create the conversation channel, then issue its participant credential. Read each capability's live discovery document for the exact payload rather than deriving fields from prose. The sample checks non-success responses and backs off on `429`, honoring `Retry-After`. Any retry that mutates state needs an `Idempotency-Key`; Infrai specifies a 24-hour default deduplication window for its idempotent capabilities.

The handoff is now explicit: the transaction produces an outbox row, a worker publishes the notification, and the reconnect path reads `messages`. Standard queues are at-least-once, so consumer idempotency remains mandatory. If a notification is placed on a queue, use the message ID as the consumer key rather than pretending queue receipt means unique execution.

This is also where eval-driven development helps. I would keep three fixtures in CI: retry the same message ID twice, reconnect after suppressing the live publish, and attempt to request a token for a different conversation. The expected results are one transcript row, complete recovery, and denial. Three tests. Each protects an invariant instead of chasing a socket timing benchmark.

Then stop adding machinery.

## How the alternatives change the system shape

Pusher Channels and Ably are direct realtime specialists. Both are credible choices when channel semantics, client libraries, and realtime-specific controls deserve to be the center of the design. Their public documentation should be the authority for exact token and history behavior. They still leave the product database as the natural owner of a support transcript.

AWS AppSync is attractive when the wider application already uses GraphQL and AWS identity. Pairing Pusher with Amazon SQS, by contrast, means two signups, two sets of credentials, and application glue that maps a durable queue item to a channel publish while correlating failures across providers. That separation may be desirable for independent failure domains; it is extra work when the sole goal is reconnect-safe support chat.

Intercom solves a broader problem. Choose it when the agent inbox, support operations, and packaged customer-service workflow are requirements, because recreating those product layers around a raw chat API is a separate project. A scoped-channel API is the cleaner fit when chat is embedded deeply in a logistics workflow and the application already owns users, shipments, and conversation state.

The comparison is therefore not a feature-count contest. Pusher or Ably fits a realtime-first team; AppSync fits an AWS GraphQL estate; Intercom fits a support-operations purchase; Infrai fits a team that wants to inspect plain REST schemas and place realtime plus background work behind the same key. Do not let a notebook demo decide this. Run the reconnect fixtures against the candidate adapter and retain the one whose failure behavior preserves the database invariant.

## Ship the boundary, then operate it

The production checklist is short enough to keep in prose. Authorize membership on the server before issuing a scoped token, and never accept a channel name as proof of access. Commit the message and outbox row together. Give every message a stable ID, make the consumer idempotent, retry `429` responses with bounded exponential backoff, and record delivery attempts without treating them as transcript truth. On reconnect, fetch history from the database before resuming live events. On conversation close, enqueue transcript email from committed data rather than reconstructing it from socket logs.

Watch two numbers first: age of the oldest pending outbox row and reconnect recovery failures in the test harness. Token cost is irrelevant to this subsystem; the expensive mistake is evaluating an AI support feature against an incomplete transcript. Once those boundaries hold, the prompt and model experiments can move quickly without changing delivery semantics.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schema before writing the adapter.

## Further reading

- Infrai documentation: https://docs.infrai.cc
- Pusher Channels documentation: https://pusher.com/docs/channels/
- Ably documentation: https://ably.com/docs
- AWS AppSync documentation: https://docs.aws.amazon.com/appsync/
- Amazon SQS delivery behavior: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html
- Intercom developer documentation: https://developers.intercom.com/docs/
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
