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
Mule 4.11 runtime), then connect a Slack app to it (see **Slack App Setup** below).

## Slack App Setup

This app is the HTTP backend for a Slack `/travel` slash command. A ready-to-use Slack
app manifest lives in the companion broker repo:
[`slack-app-manifest.yaml`](https://github.com/guhaanirban999/mulesoft-agent-fabric-travel-demo/blob/main/slack-app-manifest.yaml).

1. **Deploy this app first** and note its public base URL, e.g.
   `https://mule-slack-travel-8u1hpn.5sc6y6-4.usa-e2.cloudhub.io`. The Slack endpoint is
   that host + `/slack/travel`.
2. In the manifest, set the slash command `url` to `https://<your-app-host>/slack/travel`
   (it currently points at the deployed app above).
3. Go to <https://api.slack.com/apps> → **Create New App** → **From an app manifest**,
   pick your workspace, and paste the contents of the manifest.
4. **Install to Workspace** and approve the requested bot scopes
   (`commands`, `chat:write`, `chat:write.public`, `channels:read`, `im:write`).
5. If you change the app URL later, update the manifest (**App Manifest** page) or the
   **Slash Commands** page, then **reinstall** the app so Slack picks up the new URL.

### Using it

In any channel:

```
/travel find me flights from San Francisco to New York on June 28th 2027
```

You'll see an immediate ephemeral **"Searching for your travel options…"**, then the
flight/hotel results posted to the channel ~10–30s later once the broker responds.

> **Note:** the slash command is single-turn — Slack interactivity is disabled in the
> manifest (`interactivity.is_enabled: false`), so the app relays the broker's answer
> (or its follow-up question) as a one-shot message rather than a back-and-forth.

## Test locally

```bash
curl -i -X POST http://localhost:8081/slack/travel \
  --data-urlencode 'text=find me flights from San Francisco to New York on June 28th 2027' \
  --data-urlencode 'response_url=https://<a-callback-you-control>'
```

You get an instant `200` with the "Searching…" ACK; the flight results arrive at the
callback URL ~10–30s later.
