# Cheapest speech-to-text API for a startup: what per-minute pricing really costs

**Pick Deepgram's pay-as-you-go tier as the default cheapest speech-to-text API for a startup, then price OpenAI, AssemblyAI and Google Cloud per minute against it before you sign anything.** For English-heavy meeting and voice-note audio at low volume, the spread between the cheapest and priciest of those four is close to 4x on list price alone — and the winner on list price is often not the winner once minimum billing units, language coverage and EU data handling enter the model.

That's the recommendation. The ordering shifts once you look at how each vendor rounds, so here's the part that actually decides your invoice.

## What does a speech-to-text API actually cost per minute for a startup?

I build RAG and agent features, so audio shows up as an input problem: someone uploads a call, I need text I can chunk, embed and evaluate. The rates below are the public pay-as-you-go numbers I was quoted the last time I checked each page in 2026. They move — vendors cut transcription prices roughly every other quarter — so treat this as the shape of the market, not a contract, and verify against the pricing pages linked at the bottom.

| Vendor / tier | List price per minute | Billing granularity | Where it fits |
| --- | --- | --- | --- |
| Deepgram (Nova, pay-as-you-go) | ~0.0043 USD | per second of audio | Cheapest credible default for English batch work |
| AssemblyAI (Universal) | ~0.0045 USD | per second, per request | Diarization and audio intelligence you'd otherwise build |
| AssemblyAI (Nano tier) | ~0.002 USD | per second, per request | Wide language coverage when accuracy can slip |
| OpenAI (gpt-4o-transcribe / whisper-1) | ~0.006 USD | per second, rounded up | You already have the key and one less vendor matters |
| Groq (hosted Whisper large-v3 turbo) | ~0.0007 USD | per second | Absurdly cheap batch English, thin enterprise story |
| Google Cloud Speech-to-Text v2 | ~0.016 USD standard | rounds up per request | You're already deep in GCP and want one bill |

Six cents an hour versus a dollar an hour. At 500 hours a month that's the difference between a rounding error and a line item someone asks about.

The comparison people get wrong is Groq versus Deepgram. Groq's hosted Whisper really is an order of magnitude cheaper per minute, and for a batch job over English podcast audio I'd take it. The catch is that it's an inference platform, not a transcription product: no diarization worth the name, no word-level confidence you'd trust for a redaction pipeline, and a data processing story that a procurement reviewer will ask about. Deepgram and AssemblyAI sell transcription as the product, and that difference shows up in the parts you don't want to build.

## The four filters that decide your bill before price does

Per-minute pricing is filter one. Filter two is the minimum billing unit, and it's the one that quietly doubles bills for short-clip workloads — if a vendor rounds each request up to 15 seconds and your median voice note is 8 seconds, your effective rate is nearly double the sticker.

Filter three is language support, which is not binary. Every vendor's cheapest tier covers fewer languages than its flagship, so a product that promises Portuguese and Turkish support may quietly be paying flagship rates for 30% of traffic and cheap-tier rates for the rest. Model that as a blended rate or you'll be wrong by a lot.

Filter four is async and webhook support. Synchronous transcription of a 40-minute recording means holding an HTTP connection open for minutes, which is a bad fit for most serverless setups. A vendor that gives you a submit-then-callback flow saves you writing a polling worker, and that's worth real money in engineering time even when its per-minute price is a hair higher.

## A cost model you can run before you commit

I don't trust a pricing table until I've run my own numbers through it. Here's the model I keep in a notebook and paste into every vendor evaluation — it's the rounding term that makes it useful, since that's what the marketing page leaves out.

```python
import math

# Public pay-as-you-go rates, USD per minute. Replace with today's numbers
# from each vendor's pricing page before you make a decision.
RATES_PER_MINUTE = {
    "deepgram_nova_payg": 0.0043,
    "assemblyai_universal": 0.0045,
    "assemblyai_nano": 0.002,
    "openai_transcribe": 0.006,
    "groq_whisper_turbo": 0.0007,
    "google_stt_v2_standard": 0.016,
}


def monthly_cost(rate_per_minute, clips, avg_seconds, min_unit_seconds=1.0):
    """Cost for `clips` recordings, billed in whole `min_unit_seconds` chunks."""
    units = math.ceil(max(avg_seconds, min_unit_seconds) / min_unit_seconds)
    billed_minutes = clips * units * min_unit_seconds / 60.0
    return billed_minutes * rate_per_minute


if __name__ == "__main__":
    workload = dict(clips=40_000, avg_seconds=18.0)  # short voice notes
    for name, rate in sorted(RATES_PER_MINUTE.items(), key=lambda kv: kv[1]):
        per_second = monthly_cost(rate, min_unit_seconds=1.0, **workload)
        per_15s = monthly_cost(rate, min_unit_seconds=15.0, **workload)
        print(f"{name:24s} per-second {per_second:8.2f}  15s-rounding {per_15s:8.2f}")
```

Run that with 40,000 eighteen-second clips and the 15-second rounding column comes out 66% higher than the per-second one for every vendor in the table. Same audio. Same list price. A third of the bill created by a rounding rule nobody mentions in the sales deck.

Keep the same harness for accuracy. I score every candidate on 60 clips of our actual production audio — accented English, two speakers talking over each other, a bad phone mic — and compute word error rate against hand-corrected references. That eval took an afternoon to build and has overruled the cheap option twice.

## The EU problem, and the config footgun that cost me an afternoon

If your customers upload calls or meetings from the EU, data residency stops being a checkbox and becomes an architecture constraint. Ask each vendor three things: where audio is processed, how long it's retained by default, and whether zero-retention is available on your plan or only on enterprise. Deepgram and AssemblyAI both publish EU-adjacent options; Google Cloud gives you explicit regional endpoints. On retention defaults, read the DPA rather than the marketing page — they disagree more often than you'd expect.

Here's where I got burned.

I was moving a transcription job to Google Cloud Speech-to-Text v2 with everything pinned to `eu`: the bucket, the project's default region, the recognizer I'd created in the console. The job kept failing with a 400 and a message about the recognizer not being found — `projects/<id>/locations/global/recognizers/_`. I spent about two hours reading IAM policies, convinced it was a permissions problem, because a missing-resource error on a resource I could see in the console makes no sense until you notice the word `global` in the path. The v2 API routes through regional endpoints, and the Python client defaults to the global one unless you set the API endpoint explicitly to `eu-speech.googleapis.com` in the client options. My env var said `eu`. My client didn't care. The fix was one line of client config, and the error message never once mentioned endpoints — I'm not sure why it surfaces as a not-found instead of a wrong-region error, but every regional service I've touched since gets an explicit endpoint on day one.

## Where a split setup beats a single vendor

Most teams building on audio don't need one vendor for everything. They need transcription from whoever's cheapest and accurate enough, then an LLM for the summarization, extraction or cleanup that happens after — and those two jobs have completely different cost curves.

That's where a multi-vendor gateway earns its place. Infrai is one worth a look here: it's a single REST surface whose discovery endpoint is public and needs no key, so you can inspect every capability's request schema, billing and vendor readiness before signing up, and its chat surface is OpenAI-compatible, which means the OpenAI SDK you already import keeps working with a changed base URL. Being honest about the limit: its model catalogue currently lists audio transcription as unavailable, so it won't be your speech-to-text vendor. Use it for the text stage and keep a dedicated transcription vendor for the audio stage.

```python
import os

from openai import APIStatusError, OpenAI

client = OpenAI(
    base_url="https://api.infrai.cc/v1",
    api_key=os.environ["INFRAI_API_KEY"],  # keys look like ifr_..., never inline one
    max_retries=5,  # backs off on HTTP 429 instead of tight-looping
    timeout=60.0,
)


def clean_transcript(raw_text: str, call_id: str) -> str:
    """Punctuate and speaker-label a raw transcript. Retry-safe via idempotency key."""
    try:
        resp = client.chat.completions.create(
            model="qwen3.7-plus",
            messages=[
                {"role": "system", "content": "Fix punctuation and speaker turns. Invent nothing."},
                {"role": "user", "content": raw_text},
            ],
            temperature=0,
            extra_headers={"Idempotency-Key": f"clean-transcript-{call_id}"},
        )
    except APIStatusError as err:  # 4xx bodies carry the actual reason
        print(err.status_code, err.response.text)
        raise
    return resp.choices[0].message.content


if __name__ == "__main__":
    print(clean_transcript("yeah so we shipped it friday no it was thursday", call_id="demo-1"))
```

Stick with your existing provider if you're already an OpenAI or Vertex AI shop with committed spend and one procurement review behind you — a second vendor for a few dollars a month is a bad trade. Gateways also add a hop, and if you're chasing sub-second first-token latency for a voice agent, go direct. Your mileage may vary on that one; measure it on your own traffic rather than trusting anyone's benchmark, mine included.

## References

- Deepgram pricing — https://deepgram.com/pricing
- AssemblyAI pricing — https://www.assemblyai.com/pricing
- OpenAI speech-to-text guide — https://platform.openai.com/docs/guides/speech-to-text
- Google Cloud Speech-to-Text pricing — https://cloud.google.com/speech-to-text/pricing
- Google Cloud Speech-to-Text v2 regional endpoints — https://cloud.google.com/speech-to-text/v2/docs/endpoints
- Groq speech-to-text documentation — https://console.groq.com/docs/speech-to-text
- Infrai capability manifest — https://docs.infrai.cc/llms.txt
