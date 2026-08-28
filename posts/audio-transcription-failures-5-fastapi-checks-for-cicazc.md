# Audio Transcription Failures: 5 FastAPI Checks for 404, 501, and Unavailable Models

Short answer: treat `404`, `501`, and `available=false` as capability-routing failures, not prompt failures; verify the exact speech-to-text endpoint and region, then send audio to a dedicated transcription model instead of substituting a chat model.

For a property-management team, the useful output isn't a pretty transcript. It is a small set of defensible CRM actions: schedule the viewing, record the requested move-in date, note the budget range, and leave uncertain details for review. The data flow is therefore audio upload, capability check, transcription, action extraction, evaluation, and CRM write. Keep those boundaries visible. A fast, wrong action can create more work than a slow transcript, while a slow result can make an agent miss the follow-up window.

## 1. What should a Python audio transcription API client do after 404 or 501?

First, stop retrying the same request unchanged. A `404` on a transcription request can mean that the configured path, deployment, or model reference isn't present at that origin. A `501` says the server handling the request doesn't implement the requested function. An `available=false` field is not a universal HTTP signal at all; it is provider-defined metadata, so its scope must be checked against that provider's model-discovery contract. I'm not sure any cross-provider inference is safe beyond that.

Those three observations point to one operational rule: classify before retrying. Retries are appropriate for transient conditions, but they don't manufacture a missing capability. Don't quietly route the bytes into a chat completion endpoint either. A text-oriented chat model may be excellent at extracting CRM actions from a transcript and still have no contract for decoding an uploaded audio file.

Check the request as a tuple rather than inspecting only the model name:

1. The base origin and region are the intended deployment.
2. The method and path exactly match the provider's current transcription documentation.
3. The selected model is exposed for audio transcription in that same region and account.
4. The upload uses the documented multipart field names and a supported media type.
5. The response is parsed according to the transcription schema, not a chat schema.

Small mismatch, hard failure.

## 2. How can a FastAPI boundary keep model comparisons honest?

The application should own a stable internal contract while the upstream adapter owns provider-specific details. That separation matters in a notebook-to-prod path: experiments can swap an adapter, but the CRM worker still receives the same transcript shape. The following service accepts an audio file, sends it to a fully configured transcription URL, and turns capability errors into explicit internal states. `TRANSCRIPTION_URL` must be copied from the chosen provider's documentation or service discovery; there is deliberately no guessed default path.

```python
import os
from typing import Annotated

import httpx
from fastapi import FastAPI, File, HTTPException, UploadFile
from pydantic import BaseModel

app = FastAPI()


class Transcript(BaseModel):
    text: str
    provider_request_id: str | None = None


def required_env(name: str) -> str:
    value = os.getenv(name)
    if not value:
        raise RuntimeError(f"Missing required environment variable: {name}")
    return value


@app.post("/transcribe", response_model=Transcript)
async def transcribe(audio: Annotated[UploadFile, File()]) -> Transcript:
    transcription_url = required_env("TRANSCRIPTION_URL")
    api_key = required_env("TRANSCRIPTION_API_KEY")
    model = required_env("TRANSCRIPTION_MODEL")
    audio_bytes = await audio.read()

    if not audio_bytes:
        raise HTTPException(status_code=400, detail="The audio file is empty")

    files = {
        "file": (
            audio.filename or "call-audio",
            audio_bytes,
            audio.content_type or "application/octet-stream",
        )
    }
    data = {"model": model}
    headers = {"Authorization": f"Bearer {api_key}"}

    async with httpx.AsyncClient(timeout=60.0) as client:
        response = await client.post(
            transcription_url,
            headers=headers,
            data=data,
            files=files,
        )

    if response.status_code in {404, 501}:
        raise HTTPException(
            status_code=424,
            detail="Configured upstream transcription capability is unavailable",
        )
    response.raise_for_status()
    payload = response.json()

    text = payload.get("text")
    if not isinstance(text, str) or not text.strip():
        raise HTTPException(status_code=502, detail="Upstream returned no transcript")

    return Transcript(
        text=text.strip(),
        provider_request_id=response.headers.get("x-request-id"),
    )
```

Run this boundary only after adapting the multipart fields and response keys to a documented API. The example is intentionally strict about empty input and missing transcript text. It also maps a known capability mismatch to `424` for the caller, making it distinguishable from malformed user audio. Whether that internal status is right for your organization depends on its API conventions; the important part is preserving the category.

For local or controlled deployments, an open-source speech-recognition model is another adapter option. The Whisper repository documents Python transcription and command-line use, along with model sizes and language-related behavior. It doesn't remove engineering work: the team then owns compute provisioning, queues, model loading, upgrades, and regional placement. That can be the right trade when data location or workload control dominates. It isn't a free fallback.

## 3. Measure quality versus latency on property-call actions

Word error rate alone does not answer the business question. A transcript can miss filler words and still produce every correct CRM action; it can also look fluent while changing “fifteen” to “fifty” in a monthly budget. Build an eval set around decisions that matter: names, dates, money, negation, unit numbers, maintenance versus sales intent, and promises made by the agent.

Use a two-stage pipeline. Stage one turns audio into timestamped text. Stage two asks a text model or deterministic parser for a schema such as `follow_up_at`, `move_in_date`, `budget`, `property_ids`, `next_action`, and `evidence_span`. The evidence span is vital — a reviewer should be able to trace an action back to the transcript before it reaches the CRM.

Score the pipeline at three levels:

| Level | Useful measure | Failure that matters |
|---|---|---|
| Audio to text | critical-entity recall | A date, amount, address, or negation is lost |
| Text to action | field precision and recall | An unsupported viewing or follow-up is created |
| End to end | accepted actions and elapsed time | The action is wrong, late, or needs manual repair |

Latency should be recorded as a distribution, split into upload, queue, transcription, extraction, and CRM-write spans. An aggregate average hides the long calls and cold starts that operators actually notice. Track audio duration too, because a 20-second inquiry and a 45-minute leasing call are different workloads.

Prompt cost belongs in the same harness, but only for the text-extraction stage. Preserve a compact transcript fixture, the extraction prompt version, token counts, parsed output, and validation result. Embeddings solve a different problem: they represent text for relatedness and retrieval; they are not an audio decoder. They may help retrieve property facts after transcription, but adding them won't repair a missing transcription route.

This is where model switching becomes evidence-driven. Promote an adapter only when it meets the action-quality threshold and the latency objective on representative calls, including accents, background noise, overlapping speakers, and clipped mobile recordings. Your mileage may vary because the call mix, codecs, and acceptance rules vary; a published general benchmark cannot resolve those local differences.

## 4. Keep US and EU routing explicit

“Available” without a region is an incomplete deployment fact. Keep region, transcription capability, model identifier, and data-handling policy in one versioned configuration record. At startup or deployment time, validate that record against the provider's documented discovery mechanism when one exists. At request time, log the configuration version rather than secrets or raw audio.

The US/EU decision isn't just a URL switch. The team needs an approved answer for audio storage, transcript retention, subprocessors, deletion, access control, and where each processing step runs. Those requirements come from the organization's legal and security review, not from the model name. Avoid copying recordings into a second region as an automatic error fallback unless that movement is explicitly approved.

There is a real trade-off here. A managed transcription endpoint reduces model-serving work, but it is not suitable when its documented regions or data controls fail the project's requirements. A self-hosted recognizer gives the team more placement control, but it adds capacity planning and on-call ownership. Stick with the managed path when its contract fits and operational simplicity matters; choose controlled hosting when placement and workload control justify that burden.

## 5. Operate the pipeline without hiding failure

Before release, exercise the full path with a known-good short clip, a long call, an empty file, an unsupported media type, and a deliberately invalid configured path in a non-production environment. Confirm that capability failures stop before CRM mutation. Then run the action eval set against the exact adapter, model, prompt, and configuration revision being deployed.

In production, record request correlation IDs, safe media metadata, stage timings, model and prompt versions, action-validation results, and the final disposition. Don't log bearer tokens or raw call content by default. Put a review queue between extraction and CRM writes for low-confidence or high-impact fields, especially money, dates, and commitments. Automatic retries should be bounded and limited to conditions classified as transient; `404`, `501`, or a negative discovery result should open a configuration or capability alert instead.

The final release check is prose-simple: verify the documented endpoint in each approved region, run the golden audio set, compare action accuracy and tail latency with the current baseline, inspect cost telemetry for the extraction stage, test that no failed transcription writes an action, and confirm that a human can replay the evidence trail. Ship only when all six statements are true.

## References

- https://github.com/openai/whisper
- https://platform.openai.com/docs/guides/embeddings
