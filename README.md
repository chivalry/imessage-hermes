# imessage-hermes

> **⚠️ ABANDONED — do not use as a how-to.**
>
> This repo documents an attempt to route iMessage through a BlueBubbles +
> reverse-SSH-tunnel bridge into a [Hermes](https://github.com/NousResearch/hermes)
> agent on a remote VPS. The setup almost works, but hits a hard architectural
> wall that makes it impractical for the most common use case (one person
> messaging their own agent). See [`POSTMORTEM.md`](POSTMORTEM.md) for the
> full write-up and the alternatives.
>
> Kept public because the analysis may save someone else the same dead end.

---

## What this was trying to do

Let a Hermes agent running on a remote VPS receive and respond to iMessages
sent from the user's iPhone, using the user's existing Mac as the iMessage
bridge — without exposing anything to the public internet, without using a
third-party tunnel service, and without giving the agent any access to the
Mac beyond the BlueBubbles HTTP API.

## Why it didn't work (one-line version)

The BlueBubbles gateway adapter filters out messages where `is_from_me=true`
(to prevent the agent from looping on its own outgoing messages). When the
user and the bridge Mac are signed into the **same** Apple ID, every iMessage
the user sends to themselves is marked `is_from_me=true` on the bridge, and
the adapter silently drops it. Nothing reaches the agent.

The workaround — sign a second Apple ID into Messages on the Mac — used to
be straightforward on macOS but is no longer supported (Apple removed
multi-iMessage-account support several macOS versions ago). The remaining
options (separate macOS user account, patched Hermes source, dedicated
hardware) all violate the "minimal Mac involvement" goal that motivated
the design.

See [`POSTMORTEM.md`](POSTMORTEM.md) for the full timeline, what was
tried, what was learned, and what the right alternatives are.

## What got built before the wall

Most of the infrastructure works fine — the architectural problem is
specifically about self-messaging on one Apple ID, not about the
transport. As built:

- BlueBubbles Server on the Mac (running, authenticated, on `localhost:1234`)
- autossh-driven reverse SSH tunnel from Mac → VPS, wrapped in a
  LaunchAgent for durability
- Bidirectional forwarding (`-R 1234:127.0.0.1:1234 -L 8645:127.0.0.1:8645`)
  so the VPS can both reach BlueBubbles and receive its webhooks
- VPS-side environment configured with the BlueBubbles password

The tunnel design itself (`ARCHITECTURE.md`) is sound and reusable if you
ever want to expose a different localhost service from a Mac to a VPS
without a third-party tunnel. The dead end is specifically about the
iMessage half.

## Repo contents

```
imessage-hermes/
├── README.md           # this file
├── POSTMORTEM.md       # why it was abandoned, what to do instead
├── ARCHITECTURE.md     # the tunnel design (still useful as a pattern)
├── docs/
│   └── decisions.md    # ADR-style log of choices made during planning
└── LICENSE
```

## License

MIT. See [`LICENSE`](LICENSE).
