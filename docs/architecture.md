# Masteriva architecture

One Django project served by Daphne (ASGI), so normal HTTP pages and the live-call WebSockets run in the same
process. Pages are server-rendered Django templates with HTMX; there is no JavaScript build step.

```mermaid
flowchart LR
    subgraph Browser["Browser or installed app (PWA)"]
        UI["Django templates + HTMX + Tailwind"]
        AW["AudioWorklet<br/>16-bit PCM capture"]
        SW["Service worker<br/>offline page only"]
    end

    subgraph Edge["Shared reverse proxy"]
        Caddy["Caddy<br/>HTTPS (automatic certificates)"]
    end

    subgraph App["masteriva-web container (Daphne, ASGI)"]
        Views["HTTP views<br/>Today, practice, programs, progress,<br/>team, hiring, billing"]
        Relay["WebSocket call relay<br/>Django Channels consumer"]
        Guards["Guardrails<br/>spend ledger, caps, circuit breaker,<br/>kill switch, one call per user"]
        Metrics["Metrics in code<br/>talk share, filler words,<br/>latency, rubric total"]
        Jobs["Bounded background pool<br/>scoring, units, reports"]
        TextAI["Text AI interface<br/>OpenAI or Claude"]
        Voice["Voice session interface<br/>Deepgram Voice Agent"]
        Pay["Payments interface<br/>Stripe, Paddle, Paystack"]
        Hooks["Payment webhooks<br/>HMAC-verified, deduplicated"]
    end

    subgraph Data["Private network"]
        PG[("PostgreSQL 16")]
    end

    subgraph Providers["External services"]
        DG["Deepgram Voice Agent<br/>speech-to-text, LLM, text-to-speech"]
        LLM["OpenAI / Anthropic"]
        PSP["Hosted checkout"]
        Mail["Transactional email"]
    end

    Cron["Host cron<br/>flock + timeout"]

    UI -- HTTPS --> Caddy
    AW -- "WSS: PCM audio up,<br/>voice + events down" --> Caddy
    Caddy --> Views
    Caddy --> Relay

    Relay --> Guards
    Relay --> Voice
    Voice -- "server-side socket,<br/>open only during a call" --> DG
    Relay -- "transcript, timings" --> PG
    Relay -- "call ended" --> Jobs

    Jobs --> Metrics
    Jobs --> Guards
    Jobs --> TextAI
    TextAI --> LLM
    Views --> PG
    Views --> Pay
    Pay --> PSP
    PSP -- signed events --> Hooks
    Hooks --> PG
    Views --> Mail

    Cron -- "reminders, stale-call sweep,<br/>daily report, nightly backup" --> App
    Cron -- "pg_dump, gzip" --> PG
```

## A live call, step by step

```mermaid
sequenceDiagram
    autonumber
    participant B as Browser (AudioWorklet)
    participant R as Call relay (Channels)
    participant G as Guardrails
    participant D as Deepgram Voice Agent
    participant S as Scorer (background)

    B->>R: open WebSocket for a persona and difficulty
    R->>G: one call per user? concurrent-call cap? budget reserve
    G-->>R: allowed (or a clear "try later" message)
    R->>D: connect, send Settings (persona prompt stays server-side)
    loop conversation
        B->>R: PCM audio chunks
        R->>D: audio
        D-->>R: persona voice + ConversationText + timing events
        R-->>B: voice + transcript lines + live tips
        R->>R: measure talk time, latency, silence, time limit
    end
    D-->>R: end_call(reason) after a closing line
    R->>D: close the socket immediately (billed by connection time)
    R->>G: settle the reserved spend
    R->>S: score the call
    S->>S: metrics computed in code
    S->>S: model judges rubric items + writes coaching (validated JSON)
    S-->>B: scorecard ready (in-app notification)
```

## Apps

| App | Responsibility |
| --- | --- |
| `practice` | Personas, the live-call relay, scoring, metrics, skills, difficulty, the spend ledger and guards |
| `learn` | Programs and daily units, Today, daily evaluation, next steps, real-call logging |
| `coach` | Private text and voice coaching grounded in the user's own documents |
| `notify` | In-app and browser notifications, email with one-click unsubscribe |
| `assign` | Training assignments; progress recomputed from activity only |
| `insights` | Team progress for owners, alerts, weekly team report |
| `hire` | Hiring assessments, consent, reports (web and PDF), certification and public verification |
| `billing` | Plans, hosted checkout, verified webhooks |
| `accounts`, `console` | Sign-up, verification, lockouts, CAPTCHA, the owner console |
