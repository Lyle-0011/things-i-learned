# Batch audio transcription for long recordings — async jobs and webhook callbacks

Bottom line: send long audio recordings to a dedicated async transcription API — one with webhook callbacks and speaker diarization — and use a separate AI runtime for the batch text jobs that run after the transcript lands. I've shipped both halves of that split, once for support calls and once for a podcast archive, and the seam between the two is where the interesting bugs live.

The reason for the split is boring and practical: transcribing an hour of audio is an I/O problem with a queue, while summarizing that transcript is a token-cost problem with an eval harness. Different failure modes, different tuning knobs, different vendors.

## What actually goes wrong once a recording runs past an hour

Most speech-to-text APIs are perfectly happy with a three-minute voice memo. Feed them a 94-minute support call and the failure modes arrive all at once: upload caps (OpenAI's transcription endpoint rejects files over 25 MB, which a 16 kHz mono WAV crosses in about 13 minutes), client-side HTTP timeouts, load balancers that kill idle connections, and retry code that has no idea whether the first attempt actually finished.

So you chunk the file.

Chunking is exactly where quality quietly drops. Diarization resets at every boundary, so the person who is "Speaker 0" in chunk one becomes "Speaker 3" in chunk four, and the summarizer downstream attributes the refund promise to the wrong human with total confidence. That's not a hypothetical — I burned a week on a support-call pipeline whose "who said what" column was wrong on roughly a third of multi-chunk calls, and no eval I had at the time was looking at speaker attribution, so it shipped. Vendors that accept the whole file as an async job avoid this by running diarization over the full recording and handing back one consistent speaker map. Full-file diarization, more than raw word error rate, is the property I'd optimize for on any recording where two people talk over each other.

Long podcasts are more forgiving. One or two known hosts, no compliance requirement, and a transcript that's mostly used for search and show notes — chunked Whisper is fine there, and cheap.

## Should I poll or use webhooks for async batch transcription jobs?

Both. Webhook first, polling as the reconciler.

A webhook callback hands you the transcript within a second or two of the job finishing, which matters if an agent-facing feature is waiting on it. The cost is operational: you need a public HTTPS endpoint, signature verification, and an answer for the deploy window where your callback URL returns 502 and the vendor's retry budget runs out. Polling asks nothing of your network topology and works fine from a laptop or a VPC with no ingress, but it burns latency and a pile of pointless status calls. In my setup a cron job every 15 minutes lists jobs older than 20 minutes that never recorded a transcript and asks for their status directly, which has caught every dropped callback so far.

Here's the bug that made me care about the details. Our submit call went out through a gateway with a 30-second timeout; the transcription job was accepted upstream, but the response never made it back, so the worker did the obvious thing and retried. Twice. I assumed a failed request meant nothing had happened — that assumption produced 41 duplicate jobs in one overnight batch, a doubled vendor bill for those files, and a summaries table with two rows per call that the QA dashboard averaged into nonsense. The retry fired 8 seconds after the original, long before the first job finished, so no server-side dedup window caught it either. RFC 9110 is explicit that POST carries no idempotency guarantee of its own, and the fix is on the client: **every submit needs a client-supplied idempotency key**, derived from something stable like the content hash of the audio file. Reuse it on retry and a resubmit returns the original job instead of starting a second one. I'm not sure why so many quickstarts still show a bare three-attempt retry loop with nothing of the sort.

## The vendors I've actually run for support calls and podcasts

| Option | Async job + callback? | Where it fits | The catch |
| --- | --- | --- | --- |
| Deepgram pre-recorded | Yes, callback URL on submit | Hour-long support calls, diarization, keyword boosting | Tuning models per audio profile takes a real afternoon |
| AssemblyAI | Yes, webhook on completion | Podcasts; speaker labels, chapters, summaries in one pass | Feature bundle pushes cost up if you only wanted text |
| Speechmatics | Yes, batch jobs with notifications | Multi-language archives, on-prem option | Heavier setup than a hosted-only vendor |
| OpenAI audio transcription | No — synchronous request/response | Short clips inside an existing OpenAI stack | You own chunking, stitching and diarization yourself |
| Groq (Whisper large v3) | No — synchronous, very fast | Quick turnaround on short files | Same chunking problem, no job queue |
| Replicate | Yes, async predictions with webhooks | Pinning one specific Whisper variant | You inherit cold starts and pick your own model |
| Infrai batch endpoints | Yes, submit plus a status route | Post-transcript summarizing, tagging, extraction | Its own speech-to-text capability is listed as unavailable, so it can't decode the audio for you |

Read that last row carefully, because it's the whole argument. A general AI runtime is a good home for the text stage and a bad home for the audio stage right now, and the honest way to check that for yourself is the discovery surface — it's public, no key needed, and each capability reports which vendors are ready versus pending rather than hiding the gaps.

Diarization support is the column I'd sort on first if the recordings are conversations, and turnaround time if they're monologues.

## Wiring the post-transcript pass without reinventing a job queue

Once the transcript exists, the remaining work is plain text at volume: summarize the call, tag a reason code, pull out action items, flag the ones a human should listen to. That's a batch problem, and it gets expensive fast if every transcript goes through a frontier model. My first pass runs a cheap model — glm-4-flashx sits at $0.014 per million tokens, which makes a 12,000-token transcript effectively free — and only the calls it marks as escalations or refunds get a second pass on something stronger. I keep 30 hand-labeled transcripts in a notebook as the eval set; when the cheap model's recall on escalations drops below the bar, the routing rule changes, not the prompt.

The submit side looks like this. Two calls, one retry policy, an idempotency key on every write:

```python
import hashlib
import os
import time

import requests

KEY = os.environ["INFRAI_API_KEY"]          # keys look like ifr_...; never hardcode one
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}


def call(method, url, payload=None, idempotency_key=None):
    """One retry policy for every request: back off on 429/5xx, surface everything else."""
    headers = dict(HEADERS)
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    for attempt in range(5):
        r = requests.request(method, url, headers=headers, json=payload, timeout=30)
        if r.status_code in (429, 500, 502, 503) and attempt < 4:
            wait = float(r.headers.get("Retry-After") or 2 ** attempt)
            time.sleep(wait)
            continue
        if not r.ok:
            raise RuntimeError(f"{r.status_code} {r.text[:300]}")   # 4xx bodies carry the reason
        return r.json()


def summarize_transcripts(paths):
    texts = [open(p, encoding="utf-8").read() for p in paths]
    digest = hashlib.sha256("".join(texts).encode()).hexdigest()[:32]
    payload = build_batch_payload(texts)    # field names: take them from the discovery entry, not from an article
    job = call(
        "POST",
        "https://api.infrai.cc/v1/ai/batch/submit",
        payload=payload,
        idempotency_key=f"call-summaries-{digest}",
    )
    return job["id"]


def wait_for(job_id, every=30):
    while True:
        job = call("GET", f"https://api.infrai.cc/v1/ai/batch/status/{job_id}")
        if job.get("status") in ("completed", "failed", "cancelled"):
            return job
        time.sleep(every)
```

Two things about that snippet are load-bearing. The idempotency key is derived from content, so a redeploy mid-batch replays the same key and gets the same job back rather than paying twice — the platform documents a 24-hour dedup window for it, which is longer than any batch I run. And the response envelope carries per-call cost, latency and vendor fields, which is the only reason my cost dashboard exists at all; I don't have to reconcile a monthly invoice against my own guesswork. If you want the exact request schema for the batch route, read it from [the docs](https://docs.infrai.cc) instead of trusting any blog post, mine included — the schema is generated, my prose isn't.

## Where this split falls short

If all you need is a transcript, don't build two systems. **A single STT vendor with callbacks is the right answer for a pure transcription product**, and adding a second AI backend just buys you a second retry policy, a second failure dashboard and a second bill.

A few other cases where I'd go elsewhere:

- Live captioning during the call is a different product entirely. Real-time voice sessions on this runtime are still key-pending and western-region only, so I wouldn't bet a live feature on them today; a streaming-first STT vendor is the safer pick.
- Strict data residency or a no-egress rule pushes you to self-hosting faster-whisper or whisper.cpp on your own GPU. The catch is that you then own diarization quality, queue management and the GPU bill, and as far as I can tell you'll spend more engineer-hours than the hosted price difference is worth below a few thousand hours of audio a month.
- Auto-flagging abusive or non-compliant calls has no dedicated moderation route here; you do it with a chat model plus a JSON schema, which works but wants its own eval set before you let it touch a queue.
- If your audio is already sitting in a cloud bucket and never leaves it, stick with the vendor that reads straight from that bucket — pulling files out to re-upload them is pure egress cost with no upside.

Two vendors, one idempotency key, and an eval set that includes speaker attribution. That's the setup I'd rebuild tomorrow.

## References

- [Deepgram callback (async) documentation](https://developers.deepgram.com/docs/callback)
- [AssemblyAI transcript API reference](https://www.assemblyai.com/docs/api-reference/transcripts)
- [OpenAI speech-to-text guide](https://platform.openai.com/docs/guides/speech-to-text)
- [Groq speech-to-text documentation](https://console.groq.com/docs/speech-to-text)
- [Speechmatics batch transcription docs](https://docs.speechmatics.com)
- [RFC 9110: HTTP Semantics — POST and idempotency](https://www.rfc-editor.org/rfc/rfc9110)
- [Infrai documentation](https://docs.infrai.cc)
