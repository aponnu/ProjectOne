# Virtual Me — plan review and story index

Source: [VIRTUAL_ME_BUILD_PLAN.md](../VIRTUAL_ME_BUILD_PLAN.md) (v1.0, November 2026).

That document is the approved Phase 1 build plan (sprints S0–S6, stories VME-001 through VME-503). It has no separate “Phase 0” section. Sprint S0 is server preparation. These files are the per-story implementation plans for that whole backlog. The future backlog (VME-6xx–9xx) is not planned here.

The master plan stays the architecture source. A story file wins on the decisions listed below when it disagrees with the master plan. Do not paste real profile data, interests, or tokens into Cursor. Real config lives only on the server.

Global rules for every code story: section 9 of the master plan. A story is done only when its acceptance criteria pass and, for code stories, `uv run pytest`, `uv run ruff check`, and `uv run mypy src` are green.

## How to implement one story

1. Open the story file named `VME-XXX.md` in this folder.
2. Implement only that story. Stay inside its file list.
3. Use the prompt at the bottom of the story file.
4. When finished, list how each acceptance criterion is met.

## Decisions (use these while implementing)

These close gaps in the master plan. They are not optional.

| ID | Decision |
|---|---|
| D1 | S0 order is: VME-001 without UFW, then VME-005, then the UFW half of VME-001, then VME-003, VME-004, VME-002, VME-006. Do not enable UFW before Tailscale SSH works. |
| D2 | Pin the NVIDIA driver to the 580 branch so 590 and later cannot install. Allow 580.x security updates. Confirm the Ubuntu package name on the MSI before pinning. Do not `apt-mark hold` the whole 580 metapackage in a way that freezes security updates. |
| D3 | VME-003 records driver version and the CUDA 12 constraint in `docs/ADR/ADR-010-ollama-pin.md`. VME-401 adds the Ollama image tag and digest after a GPU smoke test. |
| D4 | VME-005 is done when SSH over Tailscale works and a LAN scan shows no new open ports. The ntfy check on port 8090 belongs to VME-404. |
| D5 | VME-006 proves the gocryptfs folder locks after reboot and mounts with `allow_other`. The Compose “refuse to start without `.unlocked`” check belongs to VME-401. |
| D6 | UFW does not control Docker-published ports. Binding 8088 and 8090 to `TAILSCALE_IP`, plus the `ss` check in VME-405, is the control that matters. |
| D7 | `vme profile validate` is a library function in VME-103 and a CLI command in VME-209. |
| D8 | VME-209 ships `vme stats` and `vme new-agent` as stubs that say the command arrives in VME-312 and VME-501. |
| D9 | VME-302 returns per-feed errors in memory. VME-304 creates `feed_status`. VME-310 writes it. |
| D10 | VME-110 stores `runs.notes` as JSON. VME-311 fills Deal Scout’s note fields. |
| D11 | Router defaults stay `max_push_per_hour: 4` and `max_push_per_day: 12`. The “≤ 6 pushes a day” success metric is a tuning target after four weeks of use, not a code constant. |
| D12 | Price-drop findings use `dedup_key + ":" + price`. Same-price reposts keep the plain `dedup_key`, so the orchestrator’s 7-day dedup still drops them. |
| D13 | Intake treats an existing `finding.id` as already accepted (outbox retry). |
| D14 | Releasing `held` findings when quiet hours end is part of VME-204, not the digest job. |
| D15 | `system` findings are capped at 3 per day and do not bypass quiet hours unless `allow_urgent_in_quiet_hours` is true. |
| D16 | `drop_reason` is set only when status is `dropped`. A budget overflow that queues a digest is recorded on the route log as rule `budget`, with an empty `drop_reason`. |
| D17 | One `VME_RUNNER_TOKEN` authenticates both runner → orchestrator and orchestrator → runner in v1. |
| D18 | VME-110 reads `<data>/runner-net/overrides.json` for enabled/disabled. VME-208 writes that file. |
| D19 | ntfy actions are published as JSON. The feedback bearer token is embedded in the action, which is an accepted v1 tradeoff: that token can only call feedback and decision routes. |
| D20 | VME-303 normalizes Slickdeals two-digit `pubDate` years and treats the store/price regexes as provisional until a sanitized fixture confirms them. |
| D21 | Rules-only Deal Scout (through VME-308 without VME-306/307) must run when Ollama is down. Semantic score and judge result default to absent. |
| D22 | Squid stays on `core` and `egress`. VME-402 proves runner-net can reach Slickdeals only through the proxy. After model pulls, Ollama is recreated without `egress`. |
| D23 | VME-405 fails if a manifest `allow_domains` entry is missing from the Squid allowlist. |

## Challenges (why those decisions exist)

### Server order can lock you out

VME-001’s UFW step allows incoming traffic only on `tailscale0`, and Tailscale is VME-005. Enabling UFW first drops LAN SSH. Password login is also disabled in VME-001. Confirm key login over Tailscale, with a temporary LAN SSH allow still in place, before the default-deny rule.

VME-004 must keep the NVIDIA runtime line that VME-003’s `nvidia-ctk` writes. Run 003 before 004. VME-002’s power-limit check needs `nvidia-smi` from VME-003. Mobile GPUs often reject `-pl`; that result is success, and cooling plus Ollama limits are the fallback.

### Driver pin is right, a hard hold is too strong

The 580 branch is still the Linux branch for Pascal (GTX 1060), with security updates planned through October 2028. A blanket `apt-mark hold` on `nvidia-driver-580` also blocks those updates. Pin the branch. Record the exact package names from the MSI.

### Acceptance criteria that belong to a later story

Several “done when” lines require work that does not exist yet: the CLI (VME-103 AC4), ntfy on 8090 (VME-005), Compose entrypoint (VME-006 AC3), Ollama image tag (VME-003 step 4), `feed_status` (VME-302 AC3), `vme stats` / `vme new-agent` (VME-209). The story files split those criteria so a story can finish.

### Alert budget and dedup disagree with themselves

Section 2.3 wants at most 6 pushes a day. Appendix A and VME-204 default to 12 a day and 4 an hour. Shipping 12 keeps the written config. Tuning toward 6 happens in real use, via quiet hours, digests, and mutes.

VME-304’s price drop reuses the deal `dedup_key`. VME-202 drops that key for 7 days. A real price drop would never notify. D12 gives price drops their own key.

`drop_reason=budget` is applied to findings that are queued, not dropped. D16 keeps `drop_reason` for actual drops.

Nothing in the master plan schedules the flush of `held` findings at the end of quiet hours. That job is VME-204.

### Tokens, Docker, and the encrypted mount

Section 6.7 says runner routes want an “orchestrator” caller, and VME-110 says the runner token. D17 uses one shared runner token in v1.

Docker bypasses UFW. Publishing a port on `0.0.0.0` would be reachable even if UFW looks correct. Compose must bind to the Tailscale IP.

gocryptfs without `allow_other` is invisible to dockerd, so containers see an empty data directory. A bind mount of a missing `.unlocked` file fails Compose before any entrypoint check runs. Unlock creates the file. The entrypoint still checks it.

Squid on an internal network plus an egress network works because Squid accepts the connection on `core` and opens a new connection on `egress`. That must be tested. The manifest allowlist and `allowlist.txt` are two lists; VME-405 compares them.

ntfy JSON actions can send `Authorization`. The header form breaks on commas and emoji labels. The token will sit in the phone notification and in ntfy’s cache. The feedback token stays low privilege.

### Slickdeals RSS is alive and messy

`https://slickdeals.net/newsearch.php?mode=frontpage&searcharea=deals&searchin=first&rss=1` was still serving in 2026. Item dates look like `Sun, 12 Apr 26 17:24:19 +0000`. Parsed as year 26 or 1926, they make the “no new frontpage item for 6 hours” alert fire constantly. Descriptions look like `*Walmart [walmart.com] *has …`, which the store regex in VME-303 does not match until the asterisks are allowed. Titles often contain two prices; “first match” can be the wrong one. Capture a sanitized fixture on the server and adjust the regexes there. Do not put a personal profile in the fixture.

### GPU and memory

A 7B Q4 model is about 4.7 GB on a 6 GB GTX 1060, which is also Jellyfin’s transcode device. One model loaded, `OLLAMA_NUM_PARALLEL=1`, and no AI requirement for the S4 rules path. `runner-net` stays at 1 GB unless a measured OOM says otherwise.

## Implementation order

Sprint outcomes match section 11 of the master plan. Only the edges in D1–D10 move.

| Order | Stories | Outcome |
|---|---|---|
| S0 | [VME-001](VME-001.md) (no UFW) → [VME-005](VME-005.md) → finish [VME-001](VME-001.md) → [VME-003](VME-003.md) → [VME-004](VME-004.md) → [VME-002](VME-002.md) → [VME-006](VME-006.md) | Hardened, GPU-ready, encrypted, Tailscale-reachable server |
| S1 | [VME-101](VME-101.md) → [VME-102](VME-102.md) → [VME-104](VME-104.md) → [VME-103](VME-103.md) → [VME-105](VME-105.md) → [VME-106](VME-106.md) | Contracts, config, audit, guarded HTTP, AI gateway |
| S2 | [VME-107](VME-107.md) → [VME-108](VME-108.md) → [VME-109](VME-109.md) → [VME-110](VME-110.md), [VME-201](VME-201.md) → [VME-202](VME-202.md) → [VME-203](VME-203.md) | Runner loads agents; orchestrator stores findings; ntfy adapter works in tests |
| S3 | [VME-204](VME-204.md) → [VME-205](VME-205.md) → [VME-206](VME-206.md), [VME-208](VME-208.md) → [VME-209](VME-209.md), [VME-401](VME-401.md) → [VME-402](VME-402.md), [VME-403](VME-403.md), [VME-404](VME-404.md) | Routing, digests, feedback, CLI; Docker isolation on the server |
| S4 | [VME-301](VME-301.md) → [VME-302](VME-302.md) → [VME-303](VME-303.md) → [VME-304](VME-304.md) → [VME-305](VME-305.md) → [VME-308](VME-308.md), [VME-310](VME-310.md) | First Slickdeals alerts on the phone, rules only |
| S5 | [VME-306](VME-306.md) → [VME-307](VME-307.md) → [VME-309](VME-309.md), [VME-311](VME-311.md), [VME-405](VME-405.md), [VME-406](VME-406.md), [VME-501](VME-501.md) → [VME-502](VME-502.md) → [VME-503](VME-503.md) | Embeddings, judge, learning, privacy checks, backups, scaffolder |
| S6 | [VME-207](VME-207.md), [VME-210](VME-210.md), [VME-312](VME-312.md), [VME-407](VME-407.md) | Decisions, narrative brief, stats, update routine |

VME-201 can start once VME-103 is done. It does not have to wait for the runner. VME-502 can be written as soon as VME-108 and VME-109 exist; it grows to cover Deal Scout when that agent exists. The sprint table above is the order to finish them, not a ban on starting a later test harness early.

## Story index

| Story | Sprint | Priority | Title |
|---|---|---|---|
| [VME-001](VME-001.md) | S0 | Must | Base OS hardening |
| [VME-002](VME-002.md) | S0 | Must | Laptop-as-server configuration |
| [VME-003](VME-003.md) | S0 | Must | NVIDIA driver 580 and container toolkit |
| [VME-004](VME-004.md) | S0 | Must | Docker Engine and log rotation |
| [VME-005](VME-005.md) | S0 | Must | Tailscale access |
| [VME-006](VME-006.md) | S0 | Must | Encrypted data folder |
| [VME-101](VME-101.md) | S1 | Must | Repo bootstrap and tooling |
| [VME-102](VME-102.md) | S1 | Must | Contracts |
| [VME-103](VME-103.md) | S1 | Must | Settings and config loading |
| [VME-104](VME-104.md) | S1 | Must | Audit log |
| [VME-105](VME-105.md) | S1 | Must | SafeHttpClient |
| [VME-106](VME-106.md) | S1 | Must | LLMGateway |
| [VME-107](VME-107.md) | S2 | Must | AgentStore |
| [VME-108](VME-108.md) | S2 | Must | BaseAgent and AgentContext |
| [VME-109](VME-109.md) | S2 | Must | Registry and discovery |
| [VME-110](VME-110.md) | S2 | Must | Runner service |
| [VME-201](VME-201.md) | S2 | Must | Orchestrator skeleton, database, and auth |
| [VME-202](VME-202.md) | S3 | Must | Findings intake |
| [VME-203](VME-203.md) | S2 | Must | Notifier and ntfy adapter |
| [VME-204](VME-204.md) | S3 | Must | Router |
| [VME-205](VME-205.md) | S3 | Must | Digest builder and schedule |
| [VME-206](VME-206.md) | S3 | Must | Feedback loop |
| [VME-207](VME-207.md) | S6 | Should | Decisions framework |
| [VME-208](VME-208.md) | S3 | Must | Agent management and watchdog |
| [VME-209](VME-209.md) | S3 | Must | CLI `vme` |
| [VME-210](VME-210.md) | S6 | Could | Narrative daily brief |
| [VME-301](VME-301.md) | S4 | Must | Deal Scout skeleton |
| [VME-302](VME-302.md) | S4 | Must | Feed fetcher |
| [VME-303](VME-303.md) | S4 | Must | Parser and normalizer |
| [VME-304](VME-304.md) | S4 | Must | Dedup and price history |
| [VME-305](VME-305.md) | S4 | Must | Hard filters and rule matcher |
| [VME-306](VME-306.md) | S5 | Should | Semantic matcher |
| [VME-307](VME-307.md) | S5 | Should | AI judge |
| [VME-308](VME-308.md) | S4 | Must | Scoring and emission |
| [VME-309](VME-309.md) | S5 | Should | Feedback learning |
| [VME-310](VME-310.md) | S4 | Must | Feed health monitor |
| [VME-311](VME-311.md) | S5 | Should | Run summary |
| [VME-312](VME-312.md) | S6 | Could | Quality stats |
| [VME-401](VME-401.md) | S3 | Must | Dockerfile and Compose |
| [VME-402](VME-402.md) | S3 | Must | Squid egress proxy |
| [VME-403](VME-403.md) | S3 | Must | Host firewall for the edge network |
| [VME-404](VME-404.md) | S3 | Must | ntfy server and phone setup |
| [VME-405](VME-405.md) | S5 | Must | Privacy verification |
| [VME-406](VME-406.md) | S5 | Must | Backups and restore drill |
| [VME-407](VME-407.md) | S6 | Should | Update routine |
| [VME-501](VME-501.md) | S5 | Must | Agent template and scaffolder |
| [VME-502](VME-502.md) | S5 | Must | Agent conformance suite |
| [VME-503](VME-503.md) | S5 | Must | Agent developer guide |

## Out of scope for this folder

No application code lives here. Implementing a story is a later session. Do not add purchasing, payment, trading, posting, or messaging-other-people capabilities in any story.
