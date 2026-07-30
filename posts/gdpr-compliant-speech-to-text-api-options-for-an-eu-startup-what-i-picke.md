# GDPR-compliant speech-to-text API options for an EU startup: what I picked and why

**Pick the speech-to-text API that will pin your audio to an EU region in a signed DPA, and treat word error rate as the second filter.** Every GDPR review I've sat through with a customer's security team went the same way — nobody opened with a question about transcription quality, they asked where the audio file physically lands, who can compel access to it, how long it's retained, and whether there's a current SOC 2 Type II report behind the API. Data residency gets a startup through procurement. Accuracy keeps the app alive afterwards. Both matter, in that order, and the order surprises engineers every single time.

I got that backwards on my first build and burned three weeks.

## Which transcription API should a small EU startup pick if data residency is non-negotiable?

Three answers, depending on where you actually are today.

Pre-revenue, still proving the feature works? Run Whisper yourself on one GPU box in an EU region — Hetzner, Scaleway, OVH, or a `europe-west` instance at any hyperscaler. The weights are MIT-licensed, the audio never leaves your own VPC, and you can answer the residency question with an infrastructure diagram instead of a vendor's marketing page. It's the cheapest defensible answer available to a two-person team, and it buys you months.

Once you have paying customers and a security questionnaire on your desk, the calculus flips. Now you're being asked for artifacts you can't manufacture yourself: an audit report, a subprocessor list, a breach-notification SLA. That's the moment to buy a managed transcription API whose EU processing is contractual rather than a config flag you set — Azure AI Speech deployed to West Europe or Sweden Central, AWS Transcribe in eu-central-1, Google's EU Speech-to-Text endpoint, or an EU-headquartered vendor such as Speechmatics or Gladia. I lean toward the EU-native vendors when the buyer is a hospital, a law firm, or anyone whose own compliance officer will read the subprocessor list line by line, because "the company is in Frankfurt" ends a conversation that "the region is in Frankfurt" merely postpones.

And if you're building a general AI app where transcription is one step in a longer chain, the pragmatic move is a US-headquartered API with a residency add-on — OpenAI's `/v1/audio/transcriptions` or Gemini's audio input via Vertex AI — provided you've read exactly which plan tiers the EU processing and zero-retention options apply to. Don't take my word for the current scope; those terms change, and the trust portal is the only version that counts.

One anti-recommendation. Groq serves Whisper models faster and cheaper than anything else I've benchmarked, and Replicate is the fastest way to try five ASR models in an afternoon, but as far as I can tell neither publishes an EU-pinned processing guarantee. Prototypes only.

## The compliance questions that cut my shortlist from nine to three

The questionnaire is boring and it's also the entire job, so here's the version I now send before I write any integration code.

1. Which legal entity processes the audio, and is there a GDPR Article 28 DPA with a maintained subprocessor list I get notified about?
2. Can I pin processing to an EU region contractually, including the failover region? Failover is where residency claims quietly break.
3. What's the default retention on audio and transcripts, and is zero retention a flag, a plan, or a phone call? Thirty-day abuse-monitoring retention is common and is often a surprise to the buyer.
4. Is my audio used for model training by default, and what does opting out change about pricing or features?
5. Is the SOC 2 report Type II, and how recent is the observation window? A Type I is a snapshot of a design, not evidence the controls ran for a year.

Two vendors dropped out at question 2 and one more at question 3, which is why the shortlist shrank so fast. None of that showed up in a comparison blog post; it came out of sales calls. Budget a week for this, not an afternoon — the replies come back slowly, and the good vendors answer in writing while the ones you should avoid answer with a call.

## How the real options compare

| Option | Where the audio lands | Ops burden | The catch |
| --- | --- | --- | --- |
| Self-hosted Whisper / faster-whisper on an EU GPU | Your VPC, provably | High: GPUs, cold starts, upgrades | No SLA, no audit report, accuracy tuning is yours |
| EU-native managed (Speechmatics, Gladia) | EU by default | Low | Smaller feature surface, priced above self-hosting at volume |
| Hyperscaler ASR (Azure AI Speech, AWS Transcribe, Google STT) | The region you deploy to | Low | Region and feature coverage differ per service, check both |
| US LLM APIs (OpenAI, Gemini on Vertex AI) | US unless a residency option applies | Lowest | Residency is a plan attribute you must verify per account |
| Speed-first inference (Groq, Replicate) | US | Low | No EU pinning I could find — prototype tier only |

Accuracy differences are real but smaller than the table suggests. On 40 clips of our own support calls — French and English, phone-quality, two speakers — Whisper large-v3 running locally landed at 11.2% WER against my hand-corrected references, while the best managed vendor came in at 8.4%. That gap mattered for us because the transcript feeds a retrieval index, and misspelled product names poison retrieval far out of proportion to their WER contribution. A custom vocabulary list closed most of it. Ask every vendor whether they support one before you compare numbers, since a managed service with keyword boosting will beat a self-hosted model with none, and that's a feature difference rather than a model difference.

Diarization is the other axis worth checking early. Whisper doesn't do speaker labels at all; you bolt on WhisperX or pyannote, and pyannote's models carry their own license acceptance step that your legal team will want to look at.

## Self-hosting Whisper, and the cold start that ate my p99

Here's the part I'd want someone to have told me.

We put faster-whisper on a serverless GPU platform with scale-to-zero, because idle A10s are expensive and our traffic looked bursty in staging. Median latency in testing was 1.9 s for a 60-second clip. Great. Then real traffic arrived, and every weekday at 09:00 CET the support teams all hit "sync" at once, our replicas had been asleep since 22:00, and p99 went to 41 s while p50 stayed at 2 s. The container pull plus loading 3 GB of large-v3 weights onto the GPU took about 38 seconds, and because the queue filled during that window, the platform cold-started three more replicas that each paid the same 38-second tax. Only sustained real traffic exposes this; no load test I'd written started from a cold fleet, which is exactly why I missed it. I'm still not sure why the second and third replicas didn't reuse a cached image layer — the platform's docs say they should. Keeping one replica permanently warm and putting a queue in front cost about €90 a month and took p99 to 6 s.

```python
# transcribe_local.py — faster-whisper on an EU GPU box.
# Load the model once at import, never inside the request handler.
from faster_whisper import WhisperModel

_MODEL = WhisperModel("large-v3", device="cuda", compute_type="float16")

def transcribe(path: str, language: str | None = None) -> dict:
    segments, info = _MODEL.transcribe(
        path,
        language=language,      # skip auto-detect when you already know it
        vad_filter=True,        # drops silence; big win on call recordings
        beam_size=1,            # beam_size=5 costs ~2x latency for a small WER gain
    )
    text = " ".join(s.text.strip() for s in segments)
    return {"text": text, "language": info.language, "duration_s": info.duration}
```

The economics only work above a threshold. A warm L4-class GPU runs roughly ten times realtime with this setup, so one instance absorbs a few hundred audio hours a month — under that, managed per-minute pricing is cheaper than the GPU you're keeping warm, and it's cheaper than the week you'll spend on the autoscaling policy. Stick with a managed API until your monthly audio volume is boring and predictable.

## Wiring it into the app: evals, retries, and what leaves the box

Whatever you pick, put the eval harness in before the vendor. Twenty to fifty clips from your actual product, hand-corrected once, plus WER and latency per clip — that's a couple of hours of work, and it turns "vendor B feels better" into a number you can put in a pull request. The same harness works against a self-hosted endpoint and a managed one, because most ASR APIs now speak the OpenAI-compatible shape.

```python
# eval_wer.py — point STT_BASE_URL at any OpenAI-compatible transcription endpoint
import os
import time
from openai import OpenAI
from jiwer import wer

client = OpenAI(base_url=os.environ["STT_BASE_URL"], api_key=os.environ["STT_API_KEY"])

def transcribe(path: str) -> str:
    with open(path, "rb") as f:
        return str(client.audio.transcriptions.create(
            model=os.environ["STT_MODEL"],
            file=f,
            response_format="text",
        ))

def score(clips: list[tuple[str, str]]) -> list[dict]:
    rows = []
    for path, reference in clips:
        t0 = time.perf_counter()
        hypothesis = transcribe(path)
        rows.append({
            "clip": path,
            "wer": round(wer(reference, hypothesis), 4),
            "latency_s": round(time.perf_counter() - t0, 2),
        })
    return rows

if __name__ == "__main__":
    for row in score([("clips/support_01.wav", open("clips/support_01.txt").read())]):
        print(row)
```

Run it weekly against your golden set. Managed models get updated underneath you, and a silent 2-point WER regression is the kind of thing you want a CI job to notice rather than a customer.

Three operational details that bit me. Retries need idempotency, because a naive backoff on a 30-minute recording will happily bill you twice for the same audio — key the job by a content hash and cache the result. Log the audio duration, model id, and processing region on every call, since duration times price is the only cost model that survives contact with production, and the region field is what you'll grep when someone asks you to prove residency. And treat the transcript as untrusted input: if it flows into an LLM prompt, a caller can read instructions aloud and you've got prompt injection with an audio delivery mechanism, which is LLM01 in the OWASP list and much easier to demo than people expect.

The honest limitation of everything above: none of it makes you compliant. It makes you defensible, which is a different and more useful thing. If you're processing special-category data under Article 9 — health recordings, anything touching criminal proceedings — the self-hosted route stops being the cheap option and becomes the only one your DPO will sign off on, and your mileage may vary depending on which supervisory authority your customers answer to.

## References

- https://github.com/openai/whisper
- https://github.com/SYSTRAN/faster-whisper
- https://platform.openai.com/docs/guides/speech-to-text
- https://learn.microsoft.com/en-us/azure/ai-services/speech-service/
- https://docs.aws.amazon.com/transcribe/latest/dg/what-is.html
- https://cloud.google.com/speech-to-text/docs/endpoints
- https://eur-lex.europa.eu/eli/reg/2016/679/oj
- https://owasp.org/www-project-top-10-for-large-language-model-applications/
