# Postmortem: why this approach was abandoned

This document explains why the BlueBubbles + reverse-SSH-tunnel bridge
into a remote-VPS-hosted Hermes agent doesn't work for the single-user
self-messaging case, what alternatives exist, and what the takeaway is
for anyone considering the same path.

## The setup, briefly

- User has an iPhone, iPad, and a primary Mac, all signed into one Apple
  ID (`chivalry@mac.com`).
- Hermes agent runs on a remote DigitalOcean VPS (`avalon`), not on the
  Mac — explicitly for isolation reasons (the user does not want agent
  software running on their main machine).
- Goal: send an iMessage from the iPhone to the user's own Apple ID,
  have the bridge Mac forward it to the VPS, have Hermes reply, and see
  the reply as a normal iMessage on all signed-in devices.

## The wall

Apple does not publish an iMessage API. Every "iMessage bridge" project
relies on a Mac signed into the target Apple ID, with software on that
Mac that reads `~/Library/Messages/chat.db` and drives `Messages.app`
via AppleScript / Messages-framework calls.

For Hermes, that software is **BlueBubbles Server**, and the integration
follows the **gateway model**:

```
iMessage → BlueBubbles Server → webhook POST → Hermes agent
Hermes agent → BlueBubbles REST API → Messages.app → iMessage
```

This model assumes the agent has its own iMessage identity — a separate
Apple ID, distinct from the human user's. That's how bot-style accounts
work everywhere else (Telegram bots, Slack bots, etc.) and BlueBubbles'
adapter is built with the same assumption.

The agent's `is_from_me` filter encodes this assumption explicitly. In
`gateway/platforms/bluebubbles.py`:

```python
is_from_me = bool(
    record.get("is_from_me")
    or record.get("fromMe")
    ...
)
if is_from_me:
    return  # silently ignore
```

This filter exists to prevent the agent from infinitely responding to
its own outgoing messages. It is correct behavior in the intended
deployment.

But in the **single-Apple-ID self-messaging** deployment — user messages
their own Apple ID, agent processes those messages — every inbound
message is `is_from_me=true` (because the account that "sent" the
message is the same account that the bridge Mac is signed into). The
filter drops every message. The agent never sees anything.

## What didn't work

### Configuration

There is no config flag to disable the `is_from_me` filter. The
allowlist (`BLUEBUBBLES_ALLOWED_USERS` etc.) operates on the sender
address *after* the from-me filter has already discarded the message,
so allowlisting yourself doesn't bypass it.

### A second Apple ID for the agent

The standard workaround in the broader BlueBubbles community is to
create a separate Apple ID for the agent and sign both IDs into
Messages.app on the bridge Mac simultaneously. The user messages the
*agent's* Apple ID; from the bridge's perspective the message has
`is_from_me=false` (different account), so the filter passes.

This used to work. **It no longer does.** Apple removed multi-iMessage-
account support from macOS several versions ago. Modern macOS allows
only one Apple ID active in Messages per macOS user, tied to the
system-wide Apple Account in System Settings. The "+" / "Add Account"
button that older guides describe simply does not exist anymore.

(Confirmed empirically by trying. The Messages → Settings → iMessage
tab shows the current Apple ID's aliases but no path to add a second
distinct account.)

### Workarounds for the workaround

Each has a real cost that pushes them outside "minimal Mac involvement":

1. **Second macOS user account on the same Mac.** Create a new local
   macOS user, sign that user into the agent's Apple ID, run
   BlueBubbles in their session via Fast User Switching. Works, but
   you're now maintaining two macOS sessions on your daily-driver
   laptop, doubling the agent's permanent presence on the machine.

2. **Patch the BlueBubbles adapter to allow self-messages.** Modify
   the `is_from_me` check in Hermes source to honor a config flag.
   Workable but every Hermes update either overwrites the patch or
   produces merge conflicts; would need to be upstreamed as a PR for
   long-term sanity.

3. **Dedicated bridge hardware.** Old Mac mini or similar runs
   BlueBubbles signed into the agent's Apple ID. Clean isolation,
   but requires hardware most people don't have lying around for this.

4. **Reverse the deployment — put Hermes on the Mac.** Use the
   `apple-imessage` bundled skill (which wraps the `imsg` CLI). One
   Apple ID, no gateway, no webhooks, no tunnel. This is what the
   simpler OpenClaw integration uses under the hood. Requires Hermes
   on the Mac, which the user explicitly declined.

## What the user did instead

Switched to a non-iMessage channel. Telegram, Signal, or Slack each
give you "chat with the agent from your phone" with zero Mac
involvement: one bot creation flow, one token in `~/.hermes/.env`, no
tunnel, no second Apple ID, no LaunchAgent, no AppleScript
permissions, no Gatekeeper bypass. The iMessage *aesthetic* is nicer
but no chat functionality is lost.

## The two architectural patterns, made explicit

If you're considering iMessage integration with any AI agent (Hermes,
OpenClaw, or anything else), recognize there are two genuinely
different patterns and pick the one that matches your deployment:

|                           | Hermes on Mac                          | Hermes on remote machine               |
|---------------------------|----------------------------------------|----------------------------------------|
| **`imsg` / AppleScript**  | ✅ Trivial. One brew install. Single Apple ID works. This is the OpenClaw model. | ❌ Hermes can't call `imsg` on a remote machine without SSH-into-laptop glue. |
| **BlueBubbles gateway**   | ⚠️ Overkill — you already have native access. | ⚠️ Requires tunnel + **a second Apple ID** for self-messaging (the wall this repo hit). |

In retrospect, the "Hermes on VPS + iMessage" intersection is the
*one* cell in this table that's genuinely hard. Every other cell is
straightforward. If iMessage and remote-Hermes are both non-
negotiable for you, the path is BlueBubbles + a second Apple ID +
either a dedicated bridge Mac or a separate macOS user account on
your daily driver.

## Lessons

1. **Read the platform adapter's message-handling logic before
   committing to a design.** I planned the entire architecture from
   the config schema and the high-level docs, neither of which mention
   the `is_from_me` filter. Reading 20 lines of `bluebubbles.py`
   earlier would have caught this before any Mac changes were made.

2. **The "easy" integration in one tool (OpenClaw) doesn't mean it's
   easy in another tool (Hermes), even when both use the same
   underlying platform.** OpenClaw runs on the Mac and uses
   AppleScript; Hermes (on a VPS) uses BlueBubbles. Different
   architectures, different constraints.

3. **macOS used to support multi-iMessage-account in Messages.app and
   no longer does.** Many older guides on the internet still describe
   the "Add Account" flow as if it works. It doesn't, as of recent
   macOS versions (Sequoia 15 confirmed; possibly earlier).

4. **BlueBubbles' deprecation in Homebrew is a Gatekeeper-signing
   issue, not a project-health issue.** The cask is scheduled to be
   disabled 2026-09-01 because the BlueBubbles binary fails Apple's
   notarization check; after that, install from the .dmg at
   bluebubbles.app. The project itself is actively maintained.
