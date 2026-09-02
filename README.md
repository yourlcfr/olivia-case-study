# Olivia — Multi-Agent Backend Platform

A backend platform where one LLM router dispatches work to fourteen specialist services, each with its own scope, tools, and guardrails.

Solo project — architecture, build, security hardening, and test discipline by [Daffa Sinulingga](https://github.com/yourlcfr). 324 commits over five weeks (May–July 2026).

## Problem

A single general-purpose assistant degrades as you add capabilities. Give one agent access to email, calendars, repositories, billing, and deployment, and three things break at once: the prompt grows until instructions get ignored, tool selection becomes unreliable, and — the part that actually matters — a mistake in one domain can act on another. There is no boundary between "draft a reply" and "push to production" except the model's judgment.

I wanted the opposite property: capabilities that scale by adding services rather than lengthening a prompt, with hard limits on what each service can touch, and an approval layer that sits between intent and action.

## What I built

A FastAPI platform with 968 route registrations across fourteen specialist services, backed by PostgreSQL with pgvector. A router service classifies incoming work and dispatches to the specialist that owns that domain; each specialist has its own tool allowlist and cannot reach outside it.

An event pipeline (emit → dedup → react) ingests webhooks from GitHub, Stripe, and Sentry. Deduplication is not optional — retried webhooks are common, and an event pipeline that acts twice on one payment is worse than one that acts zero times.

Between decision and action sits an approval and override layer. Anything crossing a risk threshold stops and waits for a human, and every override is recorded. The system is designed to be interrupted.

## Engineering highlights

- **Fourteen services, one router.** 968 route registrations across 186 Python modules, ~47,500 lines. Capability grows by adding a service with its own scope, not by extending one prompt until it stops being followed.
- **869 passing tests with a pre-commit gate.** 70 test modules. The gate runs before commit, not in review — broken code does not reach history.
- **Security as structure, not review.** Parameterized SQL throughout, path-traversal defense on every filesystem boundary, HMAC-verified webhooks so unsigned payloads are rejected before parsing.
- **Event pipeline with deduplication.** Emit, dedup, react. Webhooks from GitHub, Stripe, and Sentry are idempotent by construction, because upstreams retry and retries must not double-act.
- **Approval and override governance.** Guardrails evaluate before execution; risk-gated actions require human sign-off; overrides are logged with attribution. Autonomy is bounded on purpose.
- **Per-service telemetry.** Logs and metrics are scoped per service, so a failure is traceable to one boundary instead of one large process.
- **Background-task lifecycle.** Long work runs outside the request cycle with proper startup, cancellation, and shutdown handling rather than fire-and-forget coroutines.

## Stack

| Layer | Choices |
|---|---|
| API | Python 3, FastAPI, async, background tasks |
| Data | PostgreSQL + pgvector, 41 SQL modules |
| AI | Anthropic and OpenAI SDKs, structured output, prompt engineering |
| Events | Webhook ingestion (GitHub, Stripe, Sentry), HMAC verification, dedup |
| Governance | Approval layer, override logging, risk-based gating |
| Frontend | Next.js 14, TypeScript |
| Quality | pytest (869 tests), pre-commit quality gate |
| Deploy | Docker |

## Outcome

The architecture holds under the condition it was built for: adding a capability means adding a service, not editing a prompt and hoping the model still follows the older instructions. Scope boundaries mean a failure in one domain stays there.

The 869-test suite and pre-commit gate are what make a solo project of this size survivable — refactors are verifiable rather than hopeful.

## Code availability

The implementation is private. This case study describes the architecture and engineering decisions; the source is not public.
