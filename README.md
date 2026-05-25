# imessage-hermes

Operational record for routing iMessage through a [BlueBubbles](https://bluebubbles.app) server on a Mac, over a reverse SSH tunnel, into a [Hermes](https://github.com/NousResearch/hermes) agent running on a remote VPS.

This repository is not a piece of software you install. It is the documentation, config artifacts, and rebuild instructions for one person's personal infra. The goal: if the laptop or the VPS is wiped tomorrow, the system can be re-stood-up in under an hour by following the runbook.

## Why this exists

Apple does not publish an iMessage API. Any "AI agent in iMessage" integration requires a Mac signed into the target Apple ID to sit in the loop and relay messages. The common approaches use a third-party tunnel service (Cloudflare Tunnel, ngrok, Tailscale) to expose the bridge to whichever machine the agent runs on. This project instead reuses the user's existing SSH connection to the VPS as the transport — no third-party tunnel service, nothing exposed to the public internet, no new accounts.

## Architecture (one-line version)

```
iPhone/iPad/Mac → iMessage (Apple servers) → MacBook (BlueBubbles Server, localhost:1234)
                                                  │
                                                  └── reverse SSH tunnel ──┐
                                                                            ▼
                                                                       VPS (avalon)
                                                                       localhost:1234
                                                                            │
                                                                            ▼
                                                                       Hermes gateway
                                                                       (bluebubbles platform)
```

See [`ARCHITECTURE.md`](ARCHITECTURE.md) for the detailed picture and the trade-offs that were considered and rejected.

## Components

| Where | What | Role |
|-------|------|------|
| MacBook (Apple Silicon, macOS) | BlueBubbles Server | Talks to iMessage via AppleScript / private bits; exposes an HTTP API on `localhost:1234` |
| MacBook | `autossh` + LaunchAgent | Maintains a reverse SSH tunnel to the VPS; auto-restarts on network blips and reboot |
| VPS (`avalon`, Ubuntu 24.04, DigitalOcean) | sshd | Accepts the tunnel; binds `127.0.0.1:1234` on the VPS to BlueBubbles on the laptop |
| VPS | Hermes | Reads `~/.hermes/config.yaml`, talks to `http://127.0.0.1:1234` as if BlueBubbles were local |

## Status

Setup in progress. See [`RUNBOOK.md`](RUNBOOK.md) (once it exists) for step-by-step. Current step tracker:

- [ ] BlueBubbles Server installed on MacBook
- [ ] Reverse SSH tunnel verified working manually
- [ ] LaunchAgent installed and surviving reboot
- [ ] Hermes `bluebubbles` platform block configured
- [ ] End-to-end test: iMessage from iPhone reaches Hermes and gets a reply

## Layout

```
imessage-hermes/
├── README.md           # this file
├── ARCHITECTURE.md     # the picture + decisions
├── RUNBOOK.md          # step-by-step rebuild (TBD)
├── laptop/             # everything that lives on the MacBook
│   └── (TBD: launchd plist, install notes, BlueBubbles settings export)
├── vps/                # everything that lives on the VPS
│   └── (TBD: hermes-config.example.yaml, sshd notes)
├── docs/
│   └── decisions.md    # ADR-lite log: why autossh over Tailscale, etc.
└── LICENSE
```

## Secrets

Nothing in this repo is a secret. The BlueBubbles password, GitHub PAT, and any API keys live in `.env` files outside the repo and are referenced as `<REPLACE_ME>` placeholders in any committed config.

## License

MIT. See [`LICENSE`](LICENSE).
