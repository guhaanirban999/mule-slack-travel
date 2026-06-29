# mule-slack-travel

**Repository:** https://github.com/guhaanirban999/mule-slack-travel

A Mule 4 integration app that bridges a **Slack `/travel` slash command** to a
**Travel Agent Broker** (A2A JSON-RPC 2.0) using the **async Slack response pattern**.

Slack slash commands require an HTTP 200 within **3 seconds**, but the broker is
LLM-backed and takes ~10–60s. So the app acknowledges immediately and does the slow
work on a background thread, then pushes the result to Slack's `response_url`.

```
Slack User        Slack API        mule-slack-travel        Travel Agent Broker
   │  /travel <req>   │                   │                        │
   │─────────────────>│  POST /slack/travel                        │
   │                  │──────────────────>│  HTTP 200 "Searching…" │
   │  ⏳ Searching…   │<──────────────────│  POST /travel-agent    │
   │<─────────────────│                   │───────────────────────>│
   │                  │                   │  result: completed     │
   │                  │  POST response_url│<───────────────────────│
   │  ✈️ AF101 $312   │<──────────────────│                        │
```

## Flow

`src/main/mule/mule-slack.xml` — `slack-travel-flow`:

1. **HTTP listener** `POST /slack/travel`.
2. **Parse Slack Form** — Slack sends `application/x-www-form-urlencoded`; a DataWeave
   transform converts it (`output application/dw`) into `vars.slackForm`, then `text`
   and `response_url` are extracted.
3. **`<async>` scope** — builds the A2A JSON-RPC `message/send` request, POSTs to the
   broker, and posts the answer to the Slack `response_url`. Wrapped in
   `try`/`on-error-continue` so broker failures still send a fallback message.
4. **Immediate ACK** — returns the ephemeral "Searching for your travel options…"
   response (this is the listener response, sent within milliseconds).

## Configuration

`src/main/resources/config.yaml`:

| Property | Purpose |
|---|---|
| `http.port` | HTTP listener port (CloudHub 2.0 maps 443 → 8081) |
| `broker.host` / `broker.port` | Travel Agent Broker host/port |

## Build & run

Requires **JDK 17** and Maven.

```bash
mvn clean package
```

Deploy the resulting `target/*-mule-application.jar` to CloudHub 2.0 (or run on a local
Mule 4.11 runtime). Then point the Slack app's `/travel` slash command at
`https://<your-app-host>/slack/travel` (see `slack-app-manifest.yaml` in the broker repo).

## Test locally

```bash
curl -i -X POST http://localhost:8081/slack/travel \
  --data-urlencode 'text=find me flights from San Francisco to New York on June 28th 2027' \
  --data-urlencode 'response_url=https://<a-callback-you-control>'
```

You get an instant `200` with the "Searching…" ACK; the flight results arrive at the
callback URL ~10–30s later.
