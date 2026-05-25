# Decisions log

Short ADR-style entries. Each captures *what* was decided, *when*, and *why* — so future-you (or anyone else reading this) doesn't have to reverse-engineer the reasoning.

---

## 2026-05-23 — Use BlueBubbles as the iMessage bridge

**Decision:** Use [BlueBubbles](https://bluebubbles.app) Server on the MacBook as the iMessage adapter.

**Why:** Hermes already ships a `bluebubbles` platform adapter. All iMessage bridge projects rely on the same Mac + AppleScript / private framework trick under the hood; BlueBubbles is the most actively maintained and is the one Hermes is built against. Picking anything else means writing glue.

---

## 2026-05-23 — Run BlueBubbles on the user's primary MacBook

**Decision:** The BlueBubbles Server lives on the user's main M1 MacBook Pro, signed into their primary Apple ID.

**Why:** It's the only Mac available, it's already always-on while the user is awake, and it's already signed into the Apple ID that owns the messages the agent should see. A dedicated bridge Mac would be cleaner but adds hardware.

**Accepted trade-off:** When the lid is closed on battery the Mac sleeps and the bridge pauses. The user has chosen to accept this rather than run Amphetamine. Revisit if responsiveness becomes a problem.

---

## 2026-05-23 — Reverse SSH tunnel over existing `avalon` alias instead of Cloudflare Tunnel / ngrok / Tailscale

**Decision:** Expose BlueBubbles to the VPS via a reverse SSH tunnel (`ssh -N -R 1234:127.0.0.1:1234 avalon`) wrapped in `autossh` and a LaunchAgent.

**Why:** The user already authenticates to the VPS with SSH key auth via an `~/.ssh/config` alias. The tunnel is one extra flag on a connection that already exists. No third-party tunnel service, no new account, no public exposure of the Mac. Trade-off is that we own the durability problem (autossh + LaunchAgent), but that's a well-trodden path.

**Rejected alternatives:** Cloudflare Quick Tunnel (rotating URL + third-party dep), Cloudflare named tunnel (requires domain), ngrok (rotating URL on free tier, paid for stable), Tailscale (adds a vendor for a solved problem), router port-forward (exposes Mac to public internet — hard no).

---

## 2026-05-23 — Hermes runs only on the VPS, never on the laptop

**Decision:** Hermes lives exclusively on the VPS `avalon`. The MacBook runs BlueBubbles and the tunnel client, nothing else agent-related.

**Why:** User is unwilling to grant agent software access to their primary machine. The VPS provides isolation. The MacBook's only "agent-adjacent" exposure is the BlueBubbles HTTP API, which the agent reaches only over the user-initiated tunnel.

---

## 2026-05-23 — Public GitHub repo with MIT license

**Decision:** This repo (`chivalry/imessage-hermes`) is public, MIT-licensed.

**Why:** Contains no secrets. The runbook is generic enough to be useful to others with the same setup. No downside to making it public; potential upside of being findable.

---

## 2026-05-23 — VPS-primary working clone

**Decision:** Authoritative working clone lives on the VPS at `~/imessage-hermes`. The MacBook clones on-demand when a step needs a local file (e.g. dropping the LaunchAgent plist into place).

**Why:** Hermes (running on the VPS) is the entity making most of the edits during setup. VPS-primary minimizes round-trips through the user.

---

## 2026-05-25 — Project abandoned

**Decision:** Stop pursuing the BlueBubbles + reverse-SSH-tunnel approach. Switch to a non-iMessage chat channel (Telegram or similar) for the agent.

**Why:** The BlueBubbles platform adapter unconditionally filters out messages where `is_from_me=true` to prevent agent reply loops. With user and bridge Mac on the same Apple ID, every self-message is marked `is_from_me=true` and never reaches Hermes. The historical workaround (sign a second Apple ID into Messages.app) no longer works because modern macOS allows only one iMessage account per system user. Remaining workarounds (second macOS user, patched Hermes source, dedicated hardware) all violate the original "minimal Mac involvement" goal that motivated the VPS deployment.

See [`POSTMORTEM.md`](../POSTMORTEM.md) for full reasoning, what was tried, and the right alternative patterns.

**What still has value:** The reverse-SSH-tunnel pattern in [`ARCHITECTURE.md`](../ARCHITECTURE.md) is reusable for any "expose a Mac localhost service to a VPS without a third-party tunnel" use case. The dead end is specifically the iMessage half.
