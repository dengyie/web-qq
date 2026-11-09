<!-- markdownlint-disable-file MD033 MD045 -->
<div align="center">

# WebQQ

**High-Availability Headless QQ Community Operations Console & Self-Healing NapCat Plugin Runtime**

[![License](https://img.shields.io/badge/license-MIT-green.svg?style=flat-square)](LICENSE)
[![Version](https://img.shields.io/badge/version-v1.6.5-blue.svg?style=flat-square)](package.json)
[![Node.js](https://img.shields.io/badge/Node.js-20+-339933.svg?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED.svg?style=flat-square&logo=docker&logoColor=white)](compose.yml)
[![Protocol](https://img.shields.io/badge/Protocol-OneBot_v11-orange.svg?style=flat-square)](https://11.onebot.dev/)
[![Tests](https://img.shields.io/badge/tests-48%20passed-brightgreen.svg?style=flat-square)]()

[English](./README.md) · [简体中文](./README.zh-CN.md) · [Core Features](#core-features) · [Architecture](#system-architecture) · [Quick Start](#quick-start) · [High Availability](#high-availability--self-healing) · [Configuration](#configuration-reference)

<br>

<p align="center">
  <b>Terminates frequent account bans, session deadlocks, and cumbersome manual verification for headless QQ community operations.</b><br>
  Engineered on NapCat and OneBot v11 with bounded kernel queues, zombie slot reclamation, atomic credential rotation, and multi-tier watchdog automation.<br>
  Delivers web-based group searches, de-duplicated join requests, generational caching, sliding-window anti-brute-force defense, and HA8 Cloudflare tunnel self-healing.
</p>

</div>

---

## Motivation & Architecture Comparison

Headless QQ operations and automated bots regularly suffer from sudden session dropouts, risk control lockouts, unbuffered concurrency deadlocks that trigger cascading crashes, and a lack of clean management interfaces. WebQQ resolves these bottlenecks with an enterprise-grade operational runtime:

| Evaluation Dimension | WebQQ Operations Platform | Traditional QQ Bot Plugins | Commercial Closed-Source Suite |
| :--- | :--- | :--- | :--- |
| **Architecture & Bloat** | Pure self-contained plugin + Docker compose, zero third-party dependencies | Monolithic framework with fragile external library bindings | Proprietary closed-source binaries collecting telemetry data |
| **Kernel Crash Guard** | Bounded queues + per-call timeouts + automated zombie slot reclamation | Unbounded concurrency collapsing the underlying protocol stack | Simple rate-limiting; freezes require full manual restarts |
| **Re-login Discipline** | 10s health telemetry + 30m exponential backoff; aborts on captcha/new device | Blind tight loops that rapidly trigger permanent account bans | Relies on external SMS OTP bridges with privacy risks |
| **Web Auth & Hygiene** | Seamless token-to-HttpOnly cookie exchange + CSRF origin validation + 429 lock | Plaintext tokens exposed in query strings and browser history | Weak basic passwords lacking brute-force throttling |
| **Tunnel Availability** | Cloudflare Tunnel HA8 dual-host failover; restarts only on 3 consecutive nulls | Single tunnel point-of-failure causing permanent dropouts | Requires expensive BGP enterprise leased lines |
| **Verification Suite** | 44 mock regression tests + race, queue, timeout, and relogin forensics | Zero automated tests; verified by manual ad-hoc testing | Closed black box with unobservable internal failure modes |

---

## Core Features

| Module | Feature Highlight | Technical Implementation & Advantage |
| :--- | :--- | :--- |
| **Single-Page Console** | Unified web community dashboard | Search group IDs, preview and submit join requests (`dryRun` supported), quit groups, view group registries, and dispatch templated messages. |
| **Kernel Concurrency Gate** | Bounded queue with timeouts | Enforces maximum concurrency slots on `GroupApi`/`MsgApi`; caps queue at 8 requests with a 10s ceiling; returns 503 while safely releasing slots upon promise settlement. |
| **Zombie Slot Reclamation** | Self-healing NTEvent deadlock recovery | Detects calls that neither resolve nor reject past 5x the timeout (≥60s), forcefully reclaiming frozen slots to prevent persistent 503 cascades. |
| **Seamless Token Exchange**| Token-to-Cookie sanitization | First request via `?token=` issues a secure `HttpOnly; SameSite=Lax` cookie and performs a 302 redirect to a clean URL, keeping logs sanitized. |
| **CSRF & Brute-Force Armor**| Multi-layer endpoint defense | Validates strict `Origin.host === Host` on cookie POST requests; locks client IPs for 10 minutes upon 20 failed token attempts (resets on success). |
| **Generational Group Cache**| Eliminates dirty reads during leave | Maintains a 4s TTL cache versioned by monotonic generation counters; discards stale in-flight responses if a group was quitted during retrieval. |
| **Anti-Risk Auto Re-login** | Adaptive backoff state machine | Background daemon polls login state every 10s; offline events trigger a re-login with a 30m buffer (max 3 tries) and immediately halt on captchas. |
| **HA8 Tunnel Watchdog** | Conservative high-availability daemon | Polls Cloudflare connector metrics; falls back to MainPID edge counting; only triggers a restart after 3 consecutive zero readings. |

---

## System Architecture

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        Operator Web Browser                            │
│   ├─ Minimalist SPA Dashboard (Search · Join · Leave · Send Message)   │
│   └─ HttpOnly Cookie Auth + Strict CSRF Origin + 302 Clean Redirect    │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Local Loopback :19001 or Cloudflare Tunnel
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                 WebQQ Core Plugin (napcat-plugin-joiner)               │
│  ┌──────────────────────┐ ┌──────────────────────┐ ┌────────────────┐  │
│  │ Bounded Kernel Queue │ │ Zombie Slot Watchdog │ │ Generational   │  │
│  │ Max 2 Active / Cap 8 │ │ Auto-Reclaims Frozen │ │ 4s TTL Cache   │  │
│  └──────────┬───────────┘ └──────────┬───────────┘ └────────┬───────┘  │
│             │                        │                      │          │
│             ▼                        ▼                      ▼          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │       OneBot v11 Protocol Layer & NTQQ Engine (NapCat Docker)    │  │
│  └──────────────────────────────────┬───────────────────────────────┘  │
└─────────────────────────────────────┼──────────────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     High-Availability Watchdog Fleet                   │
│  ├─ qq-relogin-watch.service: 10s health telemetry + 30m backoff       │
│  └─ cloudflared-watchdog.timer: HA8 metrics polling + 3-strike restart │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Start

### 1. Launch Container

```bash
# Prepare configuration
cp .env.example .env

# Configure WEBUI_TOKEN and QQ_ACCOUNT in .env
# Uses local loopback 127.0.0.1 by default
docker compose up -d

# Apply official plugin whitelist patch (mandatory after container rebuilds)
bash patch-whitelist.sh && docker restart qq-napcat
```

For mesh or internal network access, overlay `compose.mesh.yml` and provide `MESH_IP`:

```bash
docker compose -f compose.yml -f compose.mesh.yml up -d
```

### 2. Extract Token & Access Console

On first launch, a cryptographically secure token is written to the plugin folder:

```bash
cat napcat/plugins/napcat-plugin-joiner/token
```

Open in a browser: `http://<host>:19001/?token=<token>`.
The server plants an `HttpOnly` cookie and automatically redirects (302) to a clean, parameter-free URL.

---

## High Availability & Self-Healing

The production environment isolates resilience policies by failure domain:

| Failure Domain | Self-Healing Mechanism |
| :--- | :--- |
| **Plugin / Container** | Bounded queue rejects overflow with 503; per-call timeouts; zombie slot recovery; container configured with `restart: unless-stopped`. |
| **Session Dropout** | `qq-relogin-watch` daemon polls every 10s; triggers only when offline; enforces a 30m cooldown to avoid account bans. |
| **Tunnel Anomaly** | `cloudflared-watchdog.timer` runs every 60s; polls HA metrics; requires 3 consecutive zero readings before restarting. |
| **Rebuild Lifecycle** | `docker restart` preserves whitelist patches; full container recreations (`docker compose down && up`) require rerunning `patch-whitelist.sh`. |

### Deploying the Re-login Watchdog

```bash
# Disable legacy timer
systemctl disable --now qq-relogin.timer

# Install and enable continuous watchdog
cp systemd/qq-relogin-watch.service /etc/systemd/system/
systemctl daemon-reload && systemctl enable --now qq-relogin-watch
```

---

## Configuration Reference

| Variable | Description | Default |
| :--- | :--- | :--- |
| `WEBUI_TOKEN` | Secret access token for NapCat WebUI | Required |
| `QQ_ACCOUNT` | Bot QQ account number | Required |
| `MESH_IP` | Bind address for mesh overlay in `compose.mesh.yml` | None |
| `JOINER_ALLOW_HOSTS` | Permitted hostnames in addition to same-origin (comma-separated) | `qq.example.com` |
| `JOINER_TRUSTED_PROXIES` | Upstream proxy IPs authorized to forward `X-Forwarded-For` / `CF-Connecting-IP` | `127.0.0.1,::1` |
| `JOINER_KERNEL_MAX` | Maximum concurrent calls to the underlying NTQQ kernel | `2` |
| `JOINER_KERNEL_QUEUE_MAX` | Queue capacity for waiting kernel calls (overflow returns 503) | `8` |
| `JOINER_KERNEL_WAIT_MS` | Maximum duration a call may wait in the queue (ms) | `10000` |
| `JOINER_KERNEL_TIMEOUT_MS`| Individual kernel execution timeout (ms, 0 defaults to 15s/20s) | `0` |
| `JOINER_KERNEL_ZOMBIE_MS` | Threshold to declare a stuck call as zombie and reclaim slot (ms) | `0` (computed) |
| `JOINER_AUTH_FAIL_WINDOW_MS`| Sliding window for tracking failed authentication attempts (ms) | `600000` |
| `QQ_WATCH_INTERVAL` | Polling frequency for login health checks (seconds) | `10` |
| `QQ_RELOGIN_MIN_INTERVAL` | Minimum cooldown between consecutive account-password re-logins (seconds) | `1800` |
| `QQ_RELOGIN_MAX_TRIES` | Maximum re-login attempts allowed per dropout event | `3` |

---

## Testing & Quality Assurance

Run the comprehensive end-to-end regression test suite with a single command:

```bash
npm test
```

The suite executes 5 validation stages (~35 seconds):
1. **Mock Regressions**: 44 test cases covering authentication, CSRF, rate limits, quotas, generational caching, and recovery;
2. **Race Condition Forensics** (`race-check.mjs`): Verifies concurrent join clicks de-duplicate into a single upstream submission;
3. **Queue Forensics** (`queue-check.mjs`): Verifies rapid 503 shedding and timer release under heavy queue saturation;
4. **Timeout Forensics** (`timeout-check.mjs`): Verifies hung calls cleanly release slots upon settling;
5. **Re-login State Verification** (`relogin-check.py`): Validates backoff limits, captcha detection, and state corruption recovery against a mock WebUI.

---

## Upstream & Acknowledgments

- **NapCat / NapCatQQ**: High-performance headless QQ protocol implementation (`mlikiowa/napcat-docker`);
- **OneBot v11**: Universal protocol specification powering modern bot extensions.

---

## License

Licensed under the [MIT License](LICENSE).
