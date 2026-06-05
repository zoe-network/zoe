# Zoe Social Layer — Sovereign Peer-to-Peer Messaging

No server. No accounts. No app. Just git repos and Zoes that know how to read them.

---

## The Problem

You want to tell a friend something. You both run Zoe. Neither of you wants to hand your data to a platform to make that happen.

The social layer solves this without introducing anything new. The message bus is already there: it's git.

---

## How It Works

Each Zoe repo grows two new files:

| File | Purpose |
|------|---------|
| `SOCIAL/OUTBOX.md` | Your append-only message log. Public. Addressed to named recipients. |
| `SOCIAL/CONTACTS.yaml` | The people you know. Maps a name to their Zoe repo URL. |

You write a message to your OUTBOX. Their Zoe polls your repo, sees a message addressed to them, and tells them. That's the whole protocol.

---

## Message Format

`SOCIAL/OUTBOX.md` — append only, never edit existing entries:

```
- 2026-06-04T17:23:00Z to:larry priority:normal
  Hey, are you free Saturday? Tomatoes are coming in.

- 2026-06-05T09:10:00Z to:james priority:low
  That pull request looks good. One comment on the federation handler.
```

Fields:

| Field | Required | Values |
|-------|----------|--------|
| `to:` | Yes | Recipient short name (matches key in their CONTACTS.yaml) |
| `priority:` | No | `urgent`, `normal`, `low` (default: `normal`) |
| Body | Yes | One or more indented lines after the header |

---

## Contacts File

`SOCIAL/CONTACTS.yaml` — who you know and where their repo lives:

```yaml
contacts:
  larry:
    name: Larry Farrell
    zoe_repo: https://github.com/lfarrell/my-assistant
    outbox: SOCIAL/OUTBOX.md
    handle: larry
  james:
    name: James O'Donnell
    zoe_repo: https://github.com/jamesod/my-zoe
    outbox: SOCIAL/OUTBOX.md
    handle: james
```

Your short name (the key) is how contacts address messages to you. If your short name in their contacts is `jim`, they write `to:jim`.

---

## What Your Zoe Does

On a schedule (cron, ScheduleWakeup, or triggered by a session start), your Zoe:

1. Reads `SOCIAL/CONTACTS.yaml`
2. For each contact, fetches their `SOCIAL/OUTBOX.md` via raw GitHub URL
3. Scans for entries addressed `to:<your-short-name>` newer than the last check timestamp
4. Notifies you through your preferred interrupt channel (Telegram, MAILBOX.md, notify-send)

Your Zoe keeps a watermark — the timestamp of the last message it delivered — so it never delivers the same message twice.

```
~/.zoe/social/watermarks.yaml   <- one ISO timestamp per contact
```

---

## Privacy Model

OUTBOX.md is public if your repo is public. That's intentional.

- Messages are addressed, not encrypted. Think postcards, not sealed envelopes.
- If you need private messaging, make your repo private and grant contacts read access.
- The body is plain text. Don't put anything in a public OUTBOX you wouldn't write on a postcard.

The practical privacy guarantee: messages aren't indexed by a platform, mined for ads, or routed through anyone else's servers. They're text files in your git repo.

---

## Bootstrap: Exchanging Handles

To add someone to your contacts, you need two things:

1. Their Zoe repo URL
2. The short name they'll use to address messages to you

The exchange is out-of-band: text, email, or in person. No discovery server. No "add friend" flow. That's by design.

A Zoe handle looks like this:

```
zoe://github.com/lfarrell/my-assistant  handle:larry
```

Share yours. Ask for theirs. Add the entry to CONTACTS.yaml. Done.

---

## Notification Channels

| Channel | How |
|---------|-----|
| Telegram bot | `curl` the Bot API with message text |
| MAILBOX.md entry | Written on next session start |
| Desktop | `notify-send` on Linux |
| Email | Gmail MCP `send_message` to yourself |

Default: MAILBOX.md (no setup). Telegram is the real-time upgrade.

---

## Reference Implementation

Minimal poller (~50 lines Python). Run via cron or ScheduleWakeup.

Key logic: fetch each contact's raw OUTBOX.md via GitHub raw URL, parse lines addressed to MY_HANDLE that are newer than the watermark, call notify(), advance watermark.

Full source: `tools/zoe-social-check.py` (add to your Zoe repo)

---

## Why This Is Interesting

The social layer inherits all of git's properties:

- **Append-only by convention** — editing OUTBOX.md changes the hash; every clone sees the rewrite. Social pressure enforces immutability.
- **Offline-first** — messages are there when you pull. No push notification required.
- **Portable** — your message history is a file you own. Export, archive, migrate. No lock-in.
- **Auditable** — `git log SOCIAL/OUTBOX.md` shows every message, timestamp, and author. No shadow edits.

The hard part isn't the protocol — it's delivery latency. Git polling is not push. For sub-minute delivery, wire a Telegram bot at the notification end. The protocol stays the same; only the last mile changes.

---

## Status

- Protocol spec: this document
- Reference poller: `tools/zoe-social-check.py`
- Bootstrap exchange: out-of-band by design
- Telegram integration: see Boswell / Chloe proactive notification pattern

---

**Zoe is always free. Zoe is code, not a product. Apache 2.0.**
