# Production

> Everything you need to safely operate an MCP server beyond `localhost`: transport, auth, scaling, observability, security, deployment.

[![OAuth](https://img.shields.io/badge/OAuth-2.1-EB5424.svg)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-1.30+-425CC7.svg)](https://opentelemetry.io)
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED.svg?logo=docker&logoColor=white)](https://www.docker.com)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-Ready-326CE5.svg?logo=kubernetes&logoColor=white)](https://kubernetes.io)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_streamable_http.md](01_streamable_http.md) | Transport | The Streamable HTTP transport, in depth |
| [02_authentication.md](02_authentication.md) | Auth | OAuth 2.1 + PKCE, RFC 9728 Resource Server pattern |
| [03_scaling.md](03_scaling.md) | Scaling | Stateful vs stateless sessions, sticky routing, horizontal scale |
| [04_observability.md](04_observability.md) | Observability | OpenTelemetry tracing, structured logs, Prometheus metrics |
| [05_security.md](05_security.md) | Security | Threat model, gateways, prompt injection, sandboxing |
| [06_deployment.md](06_deployment.md) | Deployment | Containerization, the official MCP Registry, runtime concerns |

---

## Reading Order

1. **Streamable HTTP** — the transport everything else assumes
2. **Authentication** — required before exposing anything to the internet
3. **Scaling** — stateful vs stateless decisions affect everything downstream
4. **Observability** — what to instrument and why
5. **Security** — the broader threat model beyond just auth
6. **Deployment** — containers, registries, day-2 operations

---

## Prerequisites

- Read [Fundamentals: Lifecycle](../fundamentals/04_lifecycle.md) and [Transports](../fundamentals/05_transports.md) first
- A working MCP server in [Python](../python/01_getting_started.md) or [TypeScript](../typescript/01_getting_started.md) to apply this to
- Familiarity with OAuth 2.0, OpenTelemetry concepts, and container deployments
