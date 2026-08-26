# OTP Login Providers: Webhooks, Polling Status, SMS UX, Retry, and Abuse Prevention

Short answer: for a fintech OTP login or password-reset flow, match the provider's delivery status model to your integration effort, then keep expiry, retry, resend, and abuse controls in your own application. A webhook can reduce status polling, but it does not replace an idempotent state machine or a usable SMS verification UX.

The data flow is straightforward: create one challenge, ask an SMS provider to deliver a message, show a code-entry screen, and accept one successful verification before the short expiry. The hard part is everything around that line. Delivery is asynchronous, users tap resend, mobile networks reorder events, and an attacker can turn a generous retry policy into a spend or account-enumeration problem.

## What should an OTP login provider expose for SMS verification?

Start with the boundary between your application and the provider. You need a request identifier, a delivery status vocabulary, a way to correlate status with your challenge, and a documented policy for retries. “Sent” is not “delivered,” and “delivered” is not “verified.” Your application should make the last decision.

For a password-reset message with a 10-minute expiry, I would persist a challenge record similar to this:

```python
from dataclasses import dataclass
from datetime import datetime


@dataclass
class OtpChallenge:
    challenge_id: str
    user_id: str
    code_digest: str
    expires_at: datetime
    resend_count: int
    provider_message_id: str | None
    status: str
```

The code should be generated with a cryptographically secure random source, stored as a digest rather than plaintext, and compared only after checking the challenge, user, expiry, and attempt budget. The response to a reset request should not reveal whether an account exists. That keeps the public behavior stable while the internal record can carry the detail needed for support and operations.

## What does delivery status mean for an OTP login SMS integration?

A webhook gives your service an inbound notification when a provider has a status update. Polling asks for that update on your schedule. Neither mechanism tells you whether the person entered the right code; that is an application event.

Polling is often easier to introduce because it needs no inbound endpoint. It also creates an awkward clock: poll too frequently and you add needless requests, poll too slowly and the interface appears stuck. A webhook needs signature verification, replay protection, an idempotency key, and a public endpoint that can absorb duplicate deliveries. Those are real integration costs.

| Integration shape | Operational work | Best fit | Main limitation |
| --- | --- | --- | --- |
| Send plus polling | Worker, backoff, status-query budget | A team that already operates scheduled jobs | Status can lag behind the user's screen |
| Send plus webhook | Public endpoint, signature checks, replay-safe event storage | A flow that acts on delivery evidence | More inbound security and monitoring work |
| Send only | Challenge state, user-facing retry, provider request logging | A flow where delivery status does not change the decision | Limited provider-level evidence |

Keep it boring.

Do not guess.

I once treated a provider message identifier as if it were our challenge identifier. The result was a clean-looking table that could not answer a support question: which reset attempt did this status belong to? The fix was a stable internal challenge ID carried through the request metadata and a unique constraint on each provider event. Error code 409 was a useful reminder that duplicate handling is part of the design, not an edge case.

If the provider offers neither a reliable webhook nor a useful status query, model the message as “accepted for delivery” and move on. Do not hold the verification screen hostage to a delivery-status lookup. A code can arrive after the status UI has changed, and a status can arrive after the user has already verified.

## How can a safer verification UX handle repeat requests?

Retry answers “the request failed.” Resend answers “the user did not receive or find the message.” They should not share one unlimited counter. Keep an attempt limit for code entry, a resend cooldown, and a broader per-account and per-destination budget. A resend should invalidate the previous code or define precisely which active code wins; ambiguity here produces both support tickets and avoidable security gaps.

The client needs a visible countdown, a disabled resend action during the cooldown, and an honest state for “we accepted the request.” It should preserve the entered phone number carefully, handle an expired code without clearing useful context, and offer another channel only when that channel has its own threat model. Apple documents Password AutoFill for SMS codes, so the message format and the input field should be tested on supported Apple devices instead of assumed from a desktop browser. If the UI says “try again” but the server has already spent the retry budget, the user gets confused and the system gets expensive; keep that decision server-side and return a reason the client can render without exposing account details.

For the backend, a small transition function makes policy testable. It is intentionally narrow; policy belongs in code that an eval harness can exercise.

```python
from datetime import datetime, timedelta, timezone


def can_resend(challenge: OtpChallenge, now: datetime) -> bool:
    if challenge.status in {"verified", "locked", "expired"}:
        return False
    if challenge.resend_count >= 3:
        return False
    return now >= challenge.expires_at - timedelta(minutes=9, seconds=30)


def expire_if_needed(challenge: OtpChallenge, now: datetime) -> None:
    if now >= challenge.expires_at and challenge.status == "pending":
        challenge.status = "expired"
```

The numbers in this example are policy placeholders, not universal defaults. Tune them against your threat model, delivery latency, accessibility needs, and support data. Your mileage may vary, especially across destinations and carrier routes.

## Where does abuse prevention belong in the OTP architecture?

Put controls at several identities: account, destination, IP or device signal, and provider message route. A single IP limit is easy to evade; a single phone-number limit can lock out a shared household or a legitimate corporate number. Use progressive delays, normalized destination keys, and a budget that covers both send volume and verification attempts.

Log event IDs, challenge IDs, coarse risk signals, provider status, latency, and the reason for each rejection. Avoid logging the OTP itself or unnecessary personal data. Metrics should distinguish accepted requests, delivered messages when that signal exists, successful verification, expired challenges, invalid attempts, and resend suppression. That split tells you whether a UX change improved completion or merely increased SMS traffic.

Email is a useful comparison because the delivery contract is still asynchronous. Amazon SES describes its sending and delivery features in its official documentation, but an email acceptance response is not a substitute for your own reset state and verification record.

## Choosing the integration boundary for a short-expiry reset flow

The right choice is the smallest interface that preserves evidence and control. Prefer a provider API with clear request IDs and status semantics when your team can operate the required webhook or polling worker. Prefer a simpler send-only boundary when delivery status is not actionable in the login screen; let the user enter the code and let your backend decide from the challenge record.

The catch is that this approach is not suitable when compliance or operations require provider-level delivery evidence in near real time. In that case, choose a status-capable integration and budget for signature validation, replay-safe storage, retries on inbound events, and monitoring. Stick with a send-only design when the extra status path would add more moving parts than your team can own and would not change a user or security decision.

Before shipping, run the flow through an eval harness: duplicate send requests, delayed delivery, two resends racing, an expired code, a correct code after an invalid attempt, a repeated webhook, and a provider timeout. I keep these cases beside the prompt and API tests because notebook-to-prod optimism is expensive. The release checklist is then prose rather than ceremony: verify the message template and auto-fill behavior, confirm no secret enters logs, inspect the resend and attempt budgets, replay a status event, and confirm that a reset cannot be completed twice.

## References

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://developer.apple.com/documentation/security/password_autofill
