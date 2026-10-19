# Deploy and Host Laminar on Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/new/template/laminar-observability?referralCode=sLjDXb&utm_medium=integration&utm_source=button&utm_campaign=laminar-observability)

[Laminar](https://laminar.sh/) is the open-source observability and evals platform for AI agents: OpenTelemetry-native tracing of every LLM call, tool call and agent step, evaluations, datasets, labeling queues and a SQL editor over all of your traces. This template runs Laminar's own app server and dashboard with ClickHouse and Postgres, behind one public URL.

## About Hosting Laminar

The stack is five services: Laminar (the public gateway), Frontend, AppServer, ClickHouse and Postgres.

- **Upstream's own images, pinned.** The dashboard and the Rust app server run from the official `ghcr.io/lmnr-ai` images at release v0.3.0, with ClickHouse 26.5 and Postgres 16, as in upstream's self-host compose. Nothing moves until you bump a tag.
- **One URL for the dashboard and the SDKs.** The Laminar service is a small Caddy gateway. It sends `/v1/*` (SDK and OpenTelemetry ingestion, authenticated with project API keys) to the app server and everything else to the dashboard. The app server itself has no public address: its internal dashboard API has no authentication of its own, so it stays on the private network.
- **A login in front of the dashboard.** Self-hosted Laminar signs anyone in with just an email address, no password. The gateway puts a password prompt in front of the dashboard, with a password generated for your deploy. Only SDK traffic with a valid project API key gets past it without one.
- **Migrations on every boot.** The dashboard runs the Postgres and ClickHouse migrations when it starts, and the app server waits for it. Upgrading is bumping the image tags.
- **Every secret generated.** Postgres and ClickHouse passwords, the session secret and the 32-byte key that encrypts stored LLM provider keys are generated per deploy. None of upstream's sample values from its `.env` file are used.
- **A quiet ClickHouse.** Upstream's ClickHouse settings are applied at start, the internal metric tables that write every few seconds are off, and logs go to Railway's log view instead of gigabytes of files on the container disk.

## Common Use Cases

- Tracing agents and LLM apps built with OpenAI, Anthropic, Gemini, LangChain, the Vercel AI SDK, browser agents or any OpenTelemetry exporter
- Filtering traces by status, latency, cost or tags, then replaying and debugging the failing step
- Running evaluations on datasets and tracking scores across prompt and model changes
- Ad-hoc analysis of cost, latency and token usage with SQL over the raw spans

## Dependencies for Laminar Hosting

- ClickHouse 26.5 for spans and analytics (included, private network only)
- Postgres 16 for users, projects and settings (included, private network only)
- Caddy 2 as the public gateway (included)

### Deployment Dependencies

- [Laminar self-hosting documentation](https://laminar.sh/docs/hosting-options)
- [Laminar on GitHub](https://github.com/lmnr-ai/lmnr)
- [Template source on GitHub](https://github.com/two-workspaces/laminar)

### Implementation Details

**First boot takes about five minutes.** The dashboard applies more than 70 ClickHouse migrations before it goes healthy, and the app server starts after it. Later boots take seconds.

**Sign in.** Open the Laminar service's Railway domain. The browser asks for a username and password first: `admin` and the generated `LAMINAR_PASSWORD`, both in the Laminar service's Variables tab. Then Laminar's own sign-in page asks only for an email address. Everyone who knows the gateway password can sign in as any email address, so share it only with people you'd give the whole instance to. Each email gets its own account and workspaces; invite teammates to a workspace from its settings.

**Send traces.** Create a project, copy a project API key from its settings, and point the SDK at your Railway URL over HTTPS. Railway's public domain carries HTTP only, so use the SDK's HTTP exporter:

```python
from lmnr import Laminar
Laminar.initialize(
    project_api_key="<project API key>",
    base_url="https://<your-app>.up.railway.app",
    http_port=443,
    force_http=True,
)
```

```ts
import { Laminar } from "@lmnr-ai/lmnr";
Laminar.initialize({
  projectApiKey: "<project API key>",
  baseUrl: "https://<your-app>.up.railway.app",
  httpPort: 443,
  forceHttp: true,
});
```

Any other OpenTelemetry exporter can send OTLP/HTTP (protobuf or JSON) to `https://<your-app>.up.railway.app/v1/traces` with the header `Authorization: Bearer <project API key>`.

**gRPC.** The app server's OTLP/gRPC port (8001) is only on the private network, where other services in the same Railway project can reach it on AppServer's private domain. It is not exposed publicly: Railway's domains carry HTTP only, and a TCP proxy would carry gRPC without TLS. HTTP ingestion covers both SDKs.

**Custom domain.** Add the domain to the Laminar service, then set `NEXTAUTH_URL`, `NEXT_PUBLIC_URL` and `PUBLIC_API_BASE_URL` on Frontend to `https://your-domain`.

**AI features (optional).** Chat with a trace and SQL with AI need an LLM: set `LLM_PROVIDER` (for example `openai`) and `LLM_API_KEY` on Frontend. LLM provider keys for other features can be added per workspace in the dashboard; they are encrypted with `AEAD_SECRET_KEY`, so never change that key after the first boot.

**What the gateway doesn't open.** Public trace share links and the `lmnr` CLI's browser login go through the dashboard, so they also hit the password prompt. Email, Slack and RabbitMQ are left out, as in upstream's lightweight compose. Upstream's anonymous usage telemetry is on by default; set `LAMINAR_TELEMETRY_DISABLED=true` on Frontend to turn it off.

**Full-text search is off.** Upstream's compose includes Quickwit for searching span inputs and outputs, but the v0.3.0 dashboard image doesn't ship the index definitions it needs to create the search index, so search stays empty even there. This template leaves Quickwit out (`QUICKWIT_ENABLED=false`) and saves the memory. Filters, SQL and everything else work. It will come back once upstream's image creates the index.

**Resources.** Measured on Railway after first boot and some use, the idle stack sits at about 1.8 GB of RAM: ClickHouse about 1.1 GB, the dashboard about 300 MB, the app server about 210 MB, Postgres about 150 MB and the gateway under 20 MB. That needs the Hobby plan or higher. Back up the ClickHouse and Postgres volumes; ClickHouse holds every span.

## Why Deploy Laminar on Railway?

Railway is a singular platform to deploy your infrastructure stack. Railway will host your infrastructure so you don't have to deal with configuration, while allowing you to vertically and horizontally scale it.

By deploying Laminar on Railway, you are one step closer to supporting a complete full-stack application with minimal burden. Host your servers, databases, AI agents, and more on Railway.
