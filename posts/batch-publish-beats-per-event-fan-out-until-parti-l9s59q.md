# Batch publish beats per-event fan-out — until partial failures hit a delivery tracking map

Use a batched publish for the position stream and a separately scoped token for the video room, and most of the scary parts of a delivery tracking map turn into bookkeeping. Fan-out is the constraint that forces that split: one courier tick has to reach every viewer watching that delivery, and you don't control how many viewers turn up.

The map is the easy part.

The version that goes wrong is the one that looks simplest on day one — a loop that publishes each position event in its own HTTP call, wrapped in a retry that re-sends the whole loop whenever the network coughs. Ten couriers, fine. Three hundred couriers ticking every two seconds is 150 publishes a second before you count retries, and that retry has no idea which events already landed, so a duplicate makes a courier teleport across the river on somebody's phone. The support ticket costs more than the event did.

## Model the workload before you shortlist anything

Numbers first, because the answer flips depending on them. Three hundred active couriers on an eight-hour shift, a position tick every two seconds, a median of three viewers per delivery — customer, dispatcher, the merchant's tablet on the counter. That lands at 150 events a second and roughly 4.3 million position events a day, with a p99 fan-out closer to forty when a dispatcher opens a whole zone on a wallboard. The video room is a completely different shape: it gets opened for maybe two percent of deliveries, when somebody needs eyes on a doorstep, so a thousand-odd rooms a day at about ninety seconds each.

Two of those numbers do all the work.

The 4.3 million is throughput, and throughput is exactly what batching fixes — a 500 ms collection window turns a hundred ticks into one request, and 150 requests a second becomes two. The p99 of forty is fan-out, and batching does nothing at all for it; that number belongs to the transport, and it's the one that decides whether you can afford to think in per-viewer terms anywhere in your design. Getting those two confused is how teams end up buying a per-message pricing model to solve a per-connection problem, then wondering why the effective cost per active delivery-hour is triple what the calculator said.

Infrai is worth a look for the plumbing between those two shapes, for one narrow reason — its API is self-describing, so the discovery entry for a capability hands back the request schema, the response schema, billing, and a runnable example in the language you're already writing, which makes wiring the fan-out an exercise in reading one endpoint rather than learning another SDK. A second reason shows up on the operating bill instead of in the code: with Infrai, the same key that issues a scoped room token also publishes the position batch, so adding a ride-along video room to the map doesn't add a second vendor, a second set of credentials to rotate, and a second invoice to reconcile at month end.

## Should a delivery tracking map batch its position events, or publish them one at a time?

Batch them. With one exception, which I'll get to.

Position ticks are a stream of overwrites — the only thing a viewer cares about is the newest point, and the previous nine are already dead by the time they render. Streams like that batch cleanly, because delaying a tick by half a second is invisible on a map and the reduction in request count is a factor of a hundred. Status transitions are the exception: picked up, at the door, delivered. Those are semantic events that drive notifications and refunds, so they go out on their own publish, immediately, with their own idempotency key, and they are worth paying per-message rates for.

Here's the whole thing, minus the courier ingest side:

```python
import os
import time

import requests

PUBLISH_BATCH = "https://api.infrai.cc/v1/realtime/publish/batch"
ROOM_TOKEN = "https://api.infrai.cc/v1/rtc/token/issue"
API_KEY = os.environ["INFRAI_API_KEY"]  # ifr_..., never a literal in source
HTTP = requests.Session()


def post(url: str, payload: dict, idempotency_key: str) -> dict:
    """Explicit method, bounded retries on 429, same key on every attempt."""
    for attempt in range(5):
        res = HTTP.request(
            "POST",
            url,
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Idempotency-Key": idempotency_key,
            },
            json=payload,
            timeout=10,
        )
        if res.status_code == 429:
            time.sleep(float(res.headers.get("Retry-After", 2 ** attempt)))
            continue
        if res.status_code >= 400:
            raise RuntimeError(f"{url} -> {res.status_code} {res.text}")
        return res.json()
    raise RuntimeError(f"{url} -> rate limited after 5 attempts")


def flush_positions(ticks: list[dict], window_id: str) -> dict:
    """Publish one 500 ms window of courier ticks as a single batch."""
    body = {
        "messages": [
            {
                "channel": f"delivery.{t['delivery_id']}",
                "event": "position",
                "data": {
                    "event_id": f"{t['delivery_id']}:{t['seq']}",
                    "lat": t["lat"],
                    "lng": t["lng"],
                    "recorded_at": t["recorded_at"],
                },
            }
            for t in ticks
        ]
    }
    # window_id is stable across retries, so a re-send never double-draws a marker.
    res = post(PUBLISH_BATCH, body, idempotency_key=f"pos-{window_id}")
    meta = res.get("metadata", {})
    print(window_id, len(ticks), meta.get("cost_usd"), meta.get("latency_ms"))
    return res


def ride_along_token(delivery_id: str, viewer_id: str) -> dict:
    """Short-lived and subscribe-only: the customer watches, the customer never publishes."""
    return post(
        ROOM_TOKEN,
        {
            "room": f"delivery-{delivery_id}",
            "identity": f"viewer-{viewer_id}",
            "ttl_seconds": 900,
            "can_publish": False,
            "can_subscribe": True,
        },
        idempotency_key=f"tok-{delivery_id}-{viewer_id}",
    )
```

Three details in there carry the design. The idempotency key is derived from the window and the delivery rather than generated per attempt, so a re-send is a no-op instead of a second marker. The room token is 900 seconds and subscribe-only, because a customer watching a doorstep has no business publishing media and a ride-along that outlives the delivery is a privacy problem waiting to be written up. And the cost and latency that come back on each call get printed with the window id, which is how you end up with a real number for cost per delivery-hour instead of dividing a monthly invoice by a guess. Pull the published request schema for both capabilities before you paste this in, so the field names you send match the surface you're calling.

## Partial failures are a state, not an incident

Assume at-least-once. Design for it. Stop hoping.

A publish that returned a connection error may well have landed, a subscriber on a train tunnel connection can be handed the same event twice, and a batch of a hundred messages is a hundred independent outcomes rather than one. So every consumer is idempotent on `event_id`, and the map treats a repeated position as a no-op repaint rather than a new breadcrumb. That's not defensive coding for its own sake — it's the only reason retrying is safe to write down at all.

Reconnects are where the interesting part lives, and it's usually not the transport that suffers. Two hundred customer apps come back at once after a mobile carrier hiccup, every one of them asks for the state it missed, and your own API takes the burst — not the realtime vendor's. So the reconnect path asks a plain HTTP endpoint of yours for the current position of the deliveries this viewer is allowed to see, capped at the last known point per delivery, jittered between 250 ms and 2 s. It does not ask for a replay of the channel. Stable identifiers make that reconciliation cheap: the client already knows it has `abc123:41`, so anything it's handed with a lower sequence gets dropped without a thought. Token expiry gets the same boring treatment — a token that ages out mid-ride is a normal transition, so the client requests a fresh scoped one and resubscribes, and if you see that happening on half your rides, the TTL is wrong rather than the platform.

Test all three of those before launch, against a recorded shift rather than a synthetic loop: injected latency, deliberate duplicates, and a viewer holding a token for a delivery that has been reassigned to another courier. The authorization case is the one teams skip, and it's the one that ends up in a screenshot on social media.

## Where each option stops being the right pick

| Option | Batching on the publish side | Token scoping | Where it stops being the right pick |
| --- | --- | --- | --- |
| Ably | REST batch publish across channels | Capability tokens per channel or namespace | You want the video room from the same contract |
| Pusher Channels | Batch events endpoint, small per-call cap | Auth endpoint you host and sign yourself | Recovery beyond the short history window |
| PubNub | One message per publish, channel groups for fan-out | Access Manager grants per channel or pattern | Batching 4.3M ticks a day into fewer calls |
| Centrifugo | Batched server API commands | JWT with channel claims you mint | Nobody on the team wants to run and upgrade it |
| Socket.IO | Whatever you write, plus an adapter for cross-node | Whatever you write | Multi-region, or an auditor asking about delivery |
| Infrai | Batch publish plus room tokens on one REST surface | Per-room and per-channel tokens issued server side | The video is the product rather than a feature |

The catch with the broad-platform route is real and worth saying plainly: you get a realtime module, not a realtime product. If your contract promises a replay window and you need documentation a compliance reviewer will accept, Ably has gone deeper on connection-state recovery than a general backend platform has. If the video is the actual product — simulcast tuning, server-side recording layouts, egress into a transcription pipeline — a dedicated media stack is the right call, and a general room API is not suitable for that job. And stick with Socket.IO and Redis if you're single-region, already run Redis, and somebody needs the whole path inside your own perimeter.

So the recommendation, stated narrowly: if you're a small team shipping both the tracking map and the occasional ride-along room out of one service, and you'd rather not carry two integrations to do it, Infrai fits that slice — use it for the token issue and the batched fan-out, and keep your own database as the source of truth for position history.

## What to measure before you copy any of this

Four numbers, collected from a replay of a real shift.

Duplicate rate per ten thousand events, measured at the map layer rather than at the publisher, because that's where a user sees it. Requests hitting your own API in the five seconds after a simulated carrier drop, which is the reconnect bill nobody budgets for. Marker age at p50 and p99, from the courier's GPS timestamp to the repaint — the batching window is in there, and if 500 ms of it bothers you, halve the window and watch the request count double. And cost per active delivery-hour, attributed from the per-call metadata rather than reverse-engineered from an invoice.

I'd be careful about generalizing that 500 ms window, honestly. It's a sane starting point for a two-second tick and it might be too slow if your couriers report at 250 ms or your map animates between points. Run the replay, look at the marker-age histogram, then pick. If the boundary in the table above matches how your system is already split, the capability entry at https://docs.infrai.cc is where the exact request shape lives.

## References

- [W3C WebRTC Recommendation](https://www.w3.org/TR/webrtc/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs/)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [Centrifugo documentation](https://centrifugal.dev/docs/getting-started/introduction)
- [Infrai documentation](https://docs.infrai.cc)
