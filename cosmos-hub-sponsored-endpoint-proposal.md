# Rumble Fish — Proposal: Cosmos Hub Sponsored Endpoint Provider

*Response to the [Targeted RFP: Cosmos Hub Sponsored Endpoint Provider](https://forum.cosmos.network/t/targeted-rfp-cosmos-hub-sponsored-endpoint-provider/17414) · 20 Oct 2026*

---

## Executive summary

Rumble Fish will run the Cosmos Hub sponsored endpoint service and meet all nine service requirements in the RFP, with no exceptions. The service can be available within 4 weeks of the award.

We will run three synced, pruned `cosmoshub-4` nodes at two independent providers. They sit behind a height-aware HAProxy layer. The service offers CometBFT RPC with WebSocket, REST API and gRPC over TLS. It needs no API key and has CORS enabled. Rate limits are enforced at the edge and published on a public documentation page.

[Rumble Fish](https://www.rumblefish.dev/) is a blockchain and cloud engineering company in Kraków, Poland. We were founded in 2017, have 40 engineers and have delivered 57 projects. We already run production blockchain services: [Soroban Scan](https://sorobanscan.rumblefish.dev/) and a Prices API on Stellar, and rippled node infrastructure on XRPL.

Our additional delegation request covers only infrastructure and operations (see Budget). We will submit through the Ecosystem Growth Delegation form under the title "Cosmos Hub sponsored endpoints".

---

## Compliance with the service requirements

We meet all nine requirements in full, so this proposal lists no limitations.

| # | RFP requirement | Meets | How Rumble Fish delivers it |
|---|---|---|---|
| 1 | RPC + WebSocket, API, gRPC for `cosmoshub-4` over TLS; no API key; CORS with OPTIONS preflight; canonical gRPC status codes | Yes | Each interface has its own hostname (`rpc.`, `api.`, `grpc.`) with Let's Encrypt TLS. The edge sets the CORS headers and answers preflight requests with 204. gRPC is proxied over HTTP/2 end to end, so node status codes pass through unchanged. |
| 2 | At least 2 synced nodes, height-aware LB that drops nodes more than 3 blocks behind, 99.9% monthly availability | Yes | 3 nodes (2 + 1 spare) at 2 providers. Every 2 s, HAProxy health checks compare each node's `latest_block_height` with the highest height in the pool. Clients are routed to the same node by source IP hash. |
| 3 | Per-client floor: 10 rps sustained / 50 burst; 5 rps for search calls; 5 WS subscriptions; 1 broadcast/s | Yes | Limits are set at or above these floors, per source IP, with a separate counter for each call class. |
| 4 | Shared capacity of 100 rps, incl. 20 rps search; HTTP 429 + `Retry-After`; gRPC `RESOURCE_EXHAUSTED` | Yes | Load tested to 200 rps before launch. Limits are published on a public docs page. |
| 5 | At least 1 day (~15,000 blocks) of blocks and tx index on every serving node | Yes | Retention is set to 20,000 blocks (~1.3 days). |
| 6 | `indexer = "kv"`, `index-events` empty | Yes | Set in `config.toml` / `app.toml` and checked by config management on every deploy. |
| 7 | Within 3 blocks of head, incl. caches; WS events within 10 s of each block | Yes | Height-dependent calls are never cached. External probes measure head lag and event latency. |
| 8 | Upgrade alongside the validator; serve the first block after the upgrade height, at the latest within 300 blocks | Yes | Cosmovisor with binaries staged in advance on every node, plus an on-call engineer at each upgrade height. |
| 9 | External monitoring of every interface, on-call alerting, outage notice within 12 h | Yes | External probes from 2 regions, an on-call rota, notice to the service contact within 1 h and an automated public status page. |

---

## Proposed architecture

Three pruned `cosmoshub-4` nodes run behind two HAProxy edges, spread across two independent bare-metal providers in Europe. If either provider fails, the other keeps serving.

```
                          ┌───────────────────────────────┐     ┌──────────────────────┐
                     ┌───▶│ HAProxy edge, provider A      │──┬─▶│ Node 1, provider A   │
┌──────────────────┐ │    │ TLS, CORS, per-IP limits      │  │  │ RPC, WS, API, gRPC   │
│ Client apps      │ │    │ drops nodes 3+ blocks behind  │  │  └──────────────────────┘
│ wallets,         │─┤    └───────────────────────────────┘  │  ┌──────────────────────┐
│ explorers,       │ │    ┌───────────────────────────────┐  ├─▶│ Node 2, provider B   │
│ exchanges (test) │ └───▶│ HAProxy edge, provider B      │──┤  │ RPC, WS, API, gRPC   │
│ DNS: rpc/api/grpc│      │ TLS, CORS, per-IP limits      │  │  └──────────────────────┘
└──────────────────┘      │ sticky routing by IP hash     │  │  ┌──────────────────────┐
                          └───────────────────────────────┘  └─▶│ Node 3, provider A   │
                                         ▲                      │ spare, in rotation   │
                                         ┆ probes all           └──────────────────────┘
                                         ┆ interfaces             each: kv index,
                          ┌──────────────┴───────────────────────────┐  20,000 blocks
                          │ External monitoring, 2 regions           │
                          │ head lag, WS latency, errors →           │
                          │ on-call alert + public status page       │
                          └──────────────────────────────────────────┘
```

Each edge terminates TLS, applies CORS and per-IP limits, and checks node height every 2 seconds. Each node keeps the full kv index for 20,000 blocks.

- **Setup:** nodes managed with Ansible and bootstrapped from state sync or a snapshot. Cosmovisor handles upgrades; Prometheus and Grafana collect internal metrics.
- **Ongoing maintenance:** chain upgrades, OS and CometBFT patching, disk and capacity reviews, and monthly reporting.

---

## Capacity, availability and reporting

HAProxy enforces the limits with stick tables keyed on source IP. Clients over a limit get HTTP 429 with `Retry-After`, or gRPC `RESOURCE_EXHAUSTED`.

| Call class | RFP floor per client IP | Our limit per client IP |
|---|---|---|
| General RPC, API, gRPC | 10 rps sustained, burst 50 | 10 rps sustained, burst 50 |
| `tx_search`, `block_search`, `block_results`, API tx search | 5 rps | 5 rps |
| WebSocket subscriptions | 5 concurrent | 5 concurrent |
| Transaction broadcast | 1/s | 1/s |
| Shared, all clients | 100 rps incl. 20 rps search | 100 rps, tested at 2× (200 rps, 40 rps search) |

- **Availability:** 99.9% per calendar month, excluding network-wide chain halts. That allows at most 43 minutes of downtime a month.
- **Freshness:** a node more than 3 blocks behind the pool's top height is removed within one health-check cycle.
- **Monthly report:** availability per interface, head lag, p50/p95 latency, request volume, 429 counts and incidents, posted to the forum thread.
- **Outage notice:** sent to the service contact within 1 h of detection, with the impact and an estimated restoration time. The RFP allows 12 h.
- **Public status page:** automated checks of every interface, updated every minute.

---

## Team and relevant experience

Rumble Fish has 9 years of production blockchain and cloud work behind it. The RFP does not require prior public endpoint experience, and our track record covers the skills the service needs.

- **Production blockchain services:** [Soroban Scan](https://sorobanscan.rumblefish.dev/), a Stellar explorer built on Rust and ClickHouse, and a Prices API on Stellar.
- **Node infrastructure:** rippled nodes, webhooks and subscription services for XRPL clients.
- **Cloud and DevOps:** infrastructure as code, CI/CD and DevOps as a service on AWS.
- **Ecosystem grants:** three Stellar Community Fund awards (SCF #41, #42, #45).
- **Protocol work:** a [port of Merkl to Soroban](https://www.rumblefish.dev/case-studies/Porting-Merkl-to-Stellar/) for Angle Labs and the [Tari to Ethereum bridge](https://www.rumblefish.dev/case-studies/Bridging-Tari-to-Ethereum/).

| Role | Responsibility |
|---|---|
| Service owner (Marek, CEO/CTO) | Single point of contact for the ICF and the service contact |
| Infrastructure lead | Node, load balancer and upgrade operations |
| 2 on-call DevOps engineers | 24/7 rota, incident response, monthly report |

We do not yet run a Cosmos Hub validator. The award is paid as an ATOM delegation, so we will launch a `cosmoshub-4` validator before the service goes live. The RFP author confirmed on 13 Oct 2026 that this is acceptable.

---

## Timeline

The service will be live and ready for acceptance testing 4 weeks after the award.

```
 Week 1              Week 2               Week 3               Week 4
┌────────────────┐  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ Provision      │─▶│ Sync and wire    │─▶│ Test & document  │─▶│ Acceptance       │
│ Launch         │  │ Nodes synced,    │  │ Load test 200rps │  │ ICF tests every  │
│ validator      │  │ kv index on;     │  │ Monitoring,      │  │ requirement      │
│ Order 3 servers│  │ HAProxy, TLS,    │  │ status page,     │  │ Service goes live│
│ and 2 LB VMs   │  │ CORS set         │  │ docs page        │  │                  │
└────────────────┘  └──────────────────┘  └──────────────────┘  └────────┬─────────┘
                                                                         ◆ Gate: all 9
                                                                         │ requirements pass
┌────────────────────────────────────────────────────────────────────────▼─────────┐
│ ATOM delegation: 4 quarterly rounds, starting with the first round after acceptance│
└──────────────────────────────────────────────────────────────────────────────────┘
```

The one dependency outside our control is the ICF acceptance test in week 4. Per the RFP, no infrastructure is bought before selection.

---

## Budget and additional delegation

We estimate the service costs about **USD 4,950 a month (USD 59,400 a year)** to run. These are working estimates, not yet backed by provider quotes.

| Cost item | USD / month | USD / year |
|---|---|---|
| 3 bare-metal nodes (16+ cores, 128 GB RAM, 2×2 TB NVMe), 2 providers | 600 | 7,200 |
| 2 load balancer VMs, 2 providers | 100 | 1,200 |
| External monitoring, status page, alerting | 50 | 600 |
| Operations: on-call, upgrades, reporting (~0.3 FTE) | 4,200 | 50,400 |
| **Total** | **4,950** | **59,400** |

Rumble Fish absorbs the one-off setup (about 120 engineering hours) and does not charge for it.

We size the additional ATOM delegation so that our commission covers the yearly cost:

```
Delegation (ATOM) = Annual cost (USD) / (Staking APR × Commission rate × ATOM price (USD))
```

We will fix the ATOM figure using the APR and ATOM price on the submission date. It goes only in the private application form, as the RFP requires. The award runs for four quarterly rounds from the first round after the service passes testing.

---

## Risks and mitigation

| Risk | Mitigation |
|---|---|
| A provider or data centre goes down | Nodes and load balancers are split across 2 providers; the third node absorbs the loss of one node |
| A chain upgrade binary fails | Binaries are staged in advance with Cosmovisor; a node can be restored from snapshot within the 300-block window |
| Abusive traffic or DDoS | Per-IP limits, connection caps and provider-level DDoS filtering |
| Disk growth from the full kv index | Fixed 20,000-block retention; disk alerts at 70% |
| No Cosmos Hub validator yet | Validator launched in week 1, before the service goes live |

## Contact

Rumble Fish Poland sp. z o.o., Filipa Eisenberga 11/3, 31-523 Kraków, Poland · [rumblefish.dev](https://www.rumblefish.dev/)

## Sources

- [Targeted RFP: Cosmos Hub Sponsored Endpoint Provider](https://forum.cosmos.network/t/targeted-rfp-cosmos-hub-sponsored-endpoint-provider/17414), Cosmos Forum, 6 Oct 2026
- [Rumble Fish team page](https://www.rumblefish.dev/team/)
