# Zoe — Sovereign Personal AI

An open architecture for personal AI that belongs to the user, not a platform.

**License:** Apache 2.0. Always free. No tiers, no freemium.

---

## What Zoe Is

Zoe is a published architecture and a set of reference implementations for personal AI that:

- Runs on hardware the user owns
- Learns the user it serves
- Federates with sister assistants under the user's control
- Never trades personal data for capability

Zoe is not a product. There is no company. The base model is replaceable; the doctrine isn't.

---

## Architecture

A Zoe is a stack, not a model. Six layers, each with a different owner and privacy posture:

```
┌──────────────────────────────────────────────────────────────┐
│ 6. Session context        the current conversation           │ ephemeral
│ 5. Skills / tools         skill registry                     │ local + community
│ 4. User memory            files the user controls            │ local only
│ 3. Personal LoRA          distilled user adapter             │ local only
│ 2. Zoe fine-tune          community-maintained persona       │ public
│ 1. Base model             upstream (Qwen 2.5 7B at launch)   │ public
└──────────────────────────────────────────────────────────────┘
```

Two node types per user:

| Node | Where | Does | Never Does |
|------|-------|------|------------|
| **Edge** | Pi, phone, laptop, glasses | Inference, RAG, tool use, federation | Training, distillation |
| **Steward** | Workstation, home server, GPU burst | LoRA distillation, model pulls, RAG indexing | 24/7 uptime required |

One user → one steward → many edges. Training stays inside the user's privacy domain.

---

## Federation

Zoe instances coordinate through git. No custom protocol. No federation server.

Three files make it work:

| File | Purpose |
|------|---------|
| `CLAUDE.md` | Identity — who you are, how you work. Every session reads this first. |
| `MAILBOX.md` | Append-only log — what each session did and why. Inbox for the next session. |
| `CHANGELOG.md` | One line per change. The canonical timeline. |

Push your repo to GitHub. Any AI session on any machine can read it and continue your work. That's federation at the simplest possible layer.

Full specification: [federation.md](https://zoe-network.github.io/zoe/federation.md)

---

## Getting Started

The onboarding prompt is a markdown file that any Claude session can read. It walks the user through privacy settings, git setup, workspace creation, and the first practical project — step by step, adapted to the user's OS and skill level.

### What the prompt does before you run it

1. Asks three questions: your name, what you do, what you want to work on first
2. Checks your AI provider's privacy settings and helps you opt out of model training
3. Installs git if needed and creates a version-controlled workspace
4. Builds a practical artifact (field report template, PDF pipeline) based on your answer to question 3
5. Creates a `CLAUDE.md` context file from your answers
6. Pushes to GitHub so your workspace survives your laptop

### What the prompt creates on your machine

```
~/my-assistant/
├── CLAUDE.md              # Your identity and preferences
├── MAILBOX.md             # Session log for continuity
├── README.md              # Repo description
└── templates/
    └── field-report.html  # First practical template
```

### Run it

Paste this into any Claude session:

```
read this as a prompt: https://zoe-network.github.io/zoe/start.md
```

Review the source first: [start.md](docs/start.md)

---

## Sub-Projects

| Project | Description | Repo |
|---------|-------------|------|
| **Boswell** | Privacy-first meeting transcription and summarization. Local Whisper + local LLM. | [zoe-network/zoe-boswell](https://github.com/zoe-network/zoe-boswell) |
| **Rosie** | Reference Zoe container for WSL2 / podman. Under 3 GB. | [zoe-network/rosie](https://github.com/zoe-network/rosie) |

---

## Privacy

Not a policy document. Policy in code.

- Conversation text never leaves the local Zoe without the user's explicit, per-request approval
- Outbound federation queries are redacted against policy the user defined
- The distillation pipeline is local — user examples never leave the steward during training
- Burst-to-cloud is opt-in per call, redacted before transmission

Most "local AI" means inference on your machine, everything else remote. Zoe's policy is end-to-end: training, federation, storage — all inside the user's sovereignty.

---

## Roadmap

| Milestone | Target | Status |
|-----------|--------|--------|
| Onboarding seed (start.md) | 2026-04-23 | Shipped |
| Federation protocol v1 (git-based) | 2026-04-23 | Shipped |
| Rosie reference container | 2026-05-10 | In progress |
| Whitepaper | 2026-05-10 | Draft |
| Zoe fine-tune v1 (InstructLab) | 2026-05-10 | In progress |
| Personal LoRA distillation | 2026 H2 | Documented protocol |
| Cross-instance skill invocation | 2026 H2 | Planned |

---

## Contributing

Contributions are welcome under the [Apache 2.0 license](LICENSE) with DCO sign-off.

```
git commit -s -m "description of change"
```

Areas where help is needed:

- **Model training:** InstructLab datasets for Zoe persona and SE methodology
- **Skill authoring:** Reusable workflows for common tasks (field reports, research, scheduling)
- **Edge testing:** Raspberry Pi, Jetson, phone, glasses — sovereign form factors need test cases
- **Security review:** Federation redaction policy, PII handling in tracked files

File issues on this repo. Be specific.

---

## Background Reading

- [The $634 Ghost Story](https://zoe-network.github.io/zoe/enshittification.html) — why sovereign AI matters, told through a real billing dispute
- [Federation](https://zoe-network.github.io/zoe/federation.md) — how Zoe instances coordinate through git

---

## License

Apache 2.0. See [LICENSE](LICENSE).

Zoe is code, not a product. Always free.
