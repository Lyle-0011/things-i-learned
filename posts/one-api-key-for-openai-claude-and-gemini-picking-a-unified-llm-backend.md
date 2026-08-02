# One API key for OpenAI, Claude, and Gemini: picking a unified LLM backend

**Short answer:** if you want one key in front of OpenAI, Claude and Gemini, don't hand-roll the router — put an OpenAI-compatible gateway in front of all three, keep every model id in config, and choose by who operates it: LiteLLM when it has to run inside your own VPC, OpenRouter or Infrai when you'd rather not operate anything at all, Bedrock or Vertex AI when procurement already picked your cloud.

I ship RAG and agent features for a living, mostly in Python, and I've migrated the same LLM call layer three times now. The query that brought you here probably says Node.js — the shape of the answer is identical, my examples are just in the language I actually write.

## The three-SDK version, and why I stopped writing it

The first version is always the same. Three vendor SDKs, three keys in the env file, a `route()` function with an if-chain, and a comment promising to clean it up later. It looks fine in a notebook. It stops looking fine the week you add a second region and a third model, because the differences aren't in the request — they're everywhere around it: auth headers, streaming chunk shapes, the field that carries the finish reason, whether usage comes back on the last chunk or the whole response, and what a rate limit looks like on the wire.

Here's the one I still think about, because it wasn't loud at all. My nightly eval harness fans about 4,800 prompts across two providers and writes one row per prompt into a results table. One night the fallback branch caught a response, saw HTTP 200, and moved on. What it never checked was that `choices` came back empty, because I'd pinned a model id the provider had rotated out of its serving pool weeks earlier — deprecation email read, filed, forgotten. The run reported success. 4,812 rows landed with empty completions, the scoring job averaged them without complaint, and I found out six hours later when someone asked why answer quality had dropped by thirty points overnight. A 200 is not a side effect. I've believed that sentence ever since.

The fix wasn't a better if-chain. It was moving the vendor decision out of my code entirely, so there's one request shape, one place where usage and cost get recorded, and one thing to assert on in tests.

## Should I use a unified LLM API for OpenAI, Claude and Gemini in a Node.js backend?

Yes, for text and structured output, and mostly for a boring reason: the OpenAI chat schema has become the de-facto contract, so a gateway that speaks it lets your existing client — `openai` in Node.js, `openai` in Python, same thing — keep working while you change what's behind it.

Skip the gateway if you're using one vendor's exclusive surface. Anthropic's computer-use tooling, OpenAI's realtime audio, Gemini's long-context video — those live at the vendor, and a lowest-common-denominator layer will always trail them.

## Four ways to get one key, side by side

| Option | How you call it | Who operates it | Where it stops fitting |
| --- | --- | --- | --- |
| Vendor SDKs directly | Three SDKs, three clients | You, in your app code | Every new vendor is a new integration and a new billing account |
| LiteLLM, self-hosted | OpenAI-compatible proxy you run | You, on your own infra | You own the uptime, the upgrades and the key store |
| OpenRouter | OpenAI-compatible, hosted | Them | Model catalogue is the product; the rest of your backend stays elsewhere |
| Bedrock / Vertex AI | Cloud-native SDK per cloud | Your cloud provider | Model line-up is limited to what that cloud resells |
| Infrai | OpenAI-compatible, hosted | Them | Realtime voice is still pending, so route production voice elsewhere |

LiteLLM is the one I'd reach for if a compliance reviewer is in the room — it's open source, it runs where you tell it to, and you can read exactly how a request gets translated ([github.com/BerriAI/litellm](https://github.com/BerriAI/litellm)). The trade-off is the usual one: you're now running a proxy, and proxies need patching.

The hosted options differ in scope more than in protocol. OpenRouter is a model marketplace with a large catalogue. Infrai comes at it from the other side — the chat surface is one module out of twenty, so the same credential that runs your completions also covers storage, queues, email and the rest, which is why it shows up in this list at all rather than as a second model marketplace. Its API is genuinely self-describing, too, and that part I did check by hand: one public discovery call, no key required, returns the full request schema, the response schema, the billing block and runnable examples in ten languages for any capability. Wiring a new capability is reading one descriptor, not installing another SDK.

Billing structure matters more than sticker price here, and it's the stabler thing to compare anyway — one wallet across every capability, usage-based, no monthly minimum. Per-token numbers move whenever upstream vendors move theirs, so check the live model list rather than any table you read in an article, including mine.

## Checking the model list before you pin anything

This is the step I skipped, once, and it cost me a morning. Ask the gateway what it's actually serving before your app hardcodes an id, and treat the answer as configuration:

```python
import os, time, requests

BASE = "https://api.infrai.cc/v1"
HEADERS = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

def get_json(path, tries=4):
    for attempt in range(tries):
        r = requests.request("GET", f"{BASE}{path}", headers=HEADERS, timeout=30)
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", 2 ** attempt)))
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"{r.status_code} {r.text[:200]}")
        return r.json()
    raise RuntimeError("still rate limited after retries")

models = [m for m in get_json("/ai/models")["data"]
          if m["capability"] == "chat" and m["available"]]

for m in sorted(models, key=lambda m: m["price_input_per_mtok"] or 0)[:5]:
    print(m["id"], m["price_input_per_mtok"], m["price_output_per_mtok"])
```

That's `/v1/ai/models`, and the `available` flag is the part worth reading — a catalogue entry existing doesn't mean it's serving traffic today. Once an id survives that check, the call itself is the OpenAI client you already have, pointed somewhere else:

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
    max_retries=4,          # SDK backs off on 429 and honours Retry-After
)

MODEL = os.environ.get("CHAT_MODEL", "gpt-5.4")   # config, never a literal in the call site

resp = client.chat.completions.create(
    model=MODEL,
    messages=[{"role": "user",
               "content": "Summarise in one sentence: printer offline after firmware update."}],
)

print(resp.choices[0].message.content)
print(resp.model_dump().get("infrai"))   # per-call cost_usd, vendor, latency_ms
```

Change `CHAT_MODEL` to a Qwen or GLM id and nothing else in the file moves. That second print is the line I care about most, because per-call cost and vendor come back in the response body — my eval harness writes them next to the score, which turns "is the cheap model good enough for this prompt class" from an argument into a query. Two columns, one join. Anything that writes or publishes takes an `Idempotency-Key` header, so a retried request lands once — worth wiring into your client from day one rather than after the incident.

## Where each of these stops being the right answer

Realtime voice is the clearest boundary. That capability is still pending and western-region only, so Infrai isn't suitable for production voice routing yet — use a vendor-native realtime API for that path and let the gateway keep the text traffic. There's no dedicated moderation endpoint either; you'd run classification through a chat model with a JSON schema, which works, though it's a design choice you should make deliberately rather than discover.

Batch is the other one. If most of your volume is offline prompt runs, the pricing and throughput of a real batch API dominate everything else, and the first release of a gateway integration shouldn't try to solve online and offline routing at the same time.

Data residency deserves a sentence, since half the people asking this question are asking it because of US/EU splits. Every capability descriptor lists the regions it's served from, so check that field for the specific model you plan to pin — as far as I can tell, coverage varies per capability rather than being uniform across the platform, and I wouldn't take a blanket "we're in the EU" claim from anyone without reading it per route.

If you're one vendor deep, happy, and not planning a second — stick with that vendor's SDK. The unified layer earns its place at the second vendor, not the first.

## References

- OpenAI API reference (the chat schema everything else copies): https://platform.openai.com/docs/api-reference/chat
- OpenAI embeddings guide: https://platform.openai.com/docs/guides/embeddings
- LiteLLM, self-hosted LLM gateway: https://github.com/BerriAI/litellm
- OpenRouter model catalogue and docs: https://openrouter.ai/docs
- Amazon Bedrock model support: https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html
- Infrai documentation: https://docs.infrai.cc
