# Architecture

## The picture

```
┌─────────────────────────────────────────┐         ┌──────────────────────────────────┐
│  MacBook Pro 16" M1 (always logged in)  │         │     VPS "avalon" (DigitalOcean,  │
│                                         │         │            Ubuntu 24.04)         │
│  ┌───────────────────────────────────┐  │         │                                  │
│  │  Messages.app                     │  │         │  ┌────────────────────────────┐  │
│  │  (signed into Apple ID)           │  │         │  │  Hermes gateway            │  │
│  └────────────────┬──────────────────┘  │         │  │  bluebubbles platform      │  │
│                   │ AppleScript /       │         │  │  url: http://127.0.0.1:1234│  │
│                   ▼ private API         │         │  └──────────────┬─────────────┘  │
│  ┌───────────────────────────────────┐  │         │                 │ HTTP           │
│  │  BlueBubbles Server               │  │         │                 ▼                │
│  │  http://127.0.0.1:1234            │◄─┼─ tunnel ┼─►  127.0.0.1:1234 (loopback)    │
│  └────────────────▲──────────────────┘  │         │                                  │
│                   │                     │         │  ┌────────────────────────────┐  │
│  ┌────────────────┴──────────────────┐  │         │  │  sshd                      │  │
│  │  autossh (LaunchAgent)            │  │         │  │  port 22                   │  │
│  │  ssh -N -R 1234:127.0.0.1:1234    │──┼─ TCP/22 ┼─►│  AllowTcpForwarding yes    │  │
│  │      avalon                       │  │         │  └────────────────────────────┘  │
│  └───────────────────────────────────┘  │         │                                  │
└─────────────────────────────────────────┘         └──────────────────────────────────┘
                   │
                   ▼
        Apple iMessage network
                   ▲
                   │
    iPhone / iPad / other Macs talk to it
              normally
```

## Data flow for a single message

**Inbound (user → Hermes):**
1. User sends an iMessage from iPhone to their own Apple ID.
2. Apple delivers it to all signed-in devices, including the MacBook's Messages.app.
3. BlueBubbles Server (on the MacBook) sees the new message via the Messages framework and emits a webhook / makes it available via its API on `localhost:1234`.
4. The reverse SSH tunnel makes that same port reachable as `127.0.0.1:1234` on the VPS.
5. Hermes's `bluebubbles` platform adapter receives the event, hands it to the agent.

**Outbound (Hermes → user):**
1. Agent produces a reply.
2. Hermes POSTs to `http://127.0.0.1:1234/...` on the VPS.
3. Tunnel forwards to BlueBubbles on the MacBook.
4. BlueBubbles drives Messages.app to send the iMessage.
5. The reply appears in the user's Messages app on every signed-in device.

## Decisions and trade-offs

### Why a reverse SSH tunnel and not Cloudflare Tunnel / ngrok / Tailscale

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| Cloudflare Quick Tunnel | Built into BlueBubbles UI, no account needed | Rotating URL, depends on third party | Rejected — third-party dependency for something we can already do |
| Cloudflare named tunnel | Stable URL, free | Requires a domain + CF account setup | Rejected — same reason |
| ngrok | Built into BlueBubbles | Free tier rotates URL; paid for fixed | Rejected — third-party, costs money for stable URL |
| Tailscale | Clean, encrypted, stable IP | New software on both machines, third-party control plane | Rejected — adds a vendor for a problem already solved by existing SSH |
| Reverse SSH tunnel | Reuses existing `avalon` SSH alias; no new software on the VPS; nothing exposed publicly | Needs autossh + LaunchAgent on the Mac to be durable | **Chosen** |
| Port-forward router | — | Exposes the Mac to the public internet | Hard no |

The deciding factor: the user already SSHes to the VPS with key auth via the `avalon` alias. The tunnel is just one extra flag on a connection that already exists. Nothing new to trust, nothing new to expose.

### Why BlueBubbles and not OpenClaw / sms-iMessage-bot / etc.

All current "iMessage bridge" projects ultimately work the same way (Mac + AppleScript + private framework calls). BlueBubbles has the largest user base, the most active maintenance, and is what the Hermes `bluebubbles` platform adapter is built against. Picking it means using a supported, tested path rather than writing glue.

### Why a public repo

The infra has no secrets in it (`.env` files and `LICENSE`-d binary configs stay out). The runbook is generic enough to be useful to anyone with the same setup. Cost of going public: zero. Benefit: the repo is easy to share / reference.

### Sleep behavior (known limitation)

When the MacBook lid closes on battery, macOS sleeps within minutes and BlueBubbles stops processing messages until wake. The user has chosen to accept this for now (Hermes responds while at the desk). Mitigation options if it becomes a problem later: Amphetamine, `caffeinate`, or the "prevent sleep on power adapter when display is off" setting.

### macOS permission re-grants

After major macOS updates, BlueBubbles often needs Full Disk Access / Accessibility / Contacts permissions re-granted. Document this in the runbook so future-you remembers.

## What is explicitly NOT in scope

- Hosting Hermes on the MacBook (user has chosen VPS-only for isolation reasons).
- Granting the agent any access to the MacBook beyond the BlueBubbles API surface.
- Multi-user / multi-Apple-ID routing. Single Apple ID, single agent.
- High availability. If the laptop is off or the VPS is down, the bridge is down. Acceptable for personal use.
