# Zoe — Sovereign Personal AI

An open architecture for personal AI that belongs to the user, not a platform. Zoe runs on hardware you own, learns the user it serves, federates with sister assistants under your control, and never trades personal data for capability. The base model is replaceable; the doctrine isn't. No company, no product tiers, always free.

**License:** Apache 2.0.

---

## Architecture

A Zoe is a stack, not a model. Six layers, each with a different owner and privacy posture: an ephemeral **session context** sits on top of a **skills and tools** registry (local plus community contributions), which calls into **user memory** (local-only files the user controls). Beneath that, a **personal LoRA** distilled from the user's own examples (local-only) rides on top of a community-maintained **Zoe fine-tune** (public), which in turn rides on a swappable upstream **base model** (Qwen 2.5 7B at launch). Training and personalization stay inside the user's privacy domain; only the lowest two layers are public.

Two node types per user:

| Node | Where | Does | Never Does |
|------|-------|------|------------|
| **Edge** | Pi, phone, laptop, glasses | Inference, RAG, tool use, federation | Training, distillation |
| **Steward** | Workstation, home server, GPU burst | LoRA distillation, model pulls, RAG indexing | 24/7 uptime required |

One user → one steward → many edges.

---

## Mixture of Models (MoM)

MoM is the reference implementation of Zoe. Instead of routing every request to one large model, a small permanent local model acts as the human interface and dispatches work to a committee of specialists.

Four tiers:

| Tier | Role | Examples |
|------|------|----------|
| **Human Interface** | Sub-1B local model. Always on, always local. Owns the conversation, routes work. | Local router |
| **Local Tier** | Larger local models for reasoning, code, research. | Ollama-served Granite, Qwen, Llama |
| **Tool Tier** | Deterministic capability via MCP servers. | Gmail, calendar, filesystem, search |
| **Cloud Tier** | Optional API committee for hard problems. | Claude, GPT, Gemini |

The architecture degrades gracefully: fully functional offline, better with cloud access. It is designed for users who context-switch frequently and need ambient cognitive support without surrendering the conversation to a remote endpoint.

The reference implementation is **Rosie** — a sub-3 GB container.

Full architecture: [MoM canonical doc](https://zoe-network.github.io/zoe-boswell/mom.html)

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

The onboarding prompt is a markdown file any capable AI session can read. It walks the user through privacy settings, git setup, workspace creation, and a first practical project — adapted to the user's OS and skill level.

### What the prompt does

1. Asks three questions: your name, what you do, what you want to work on first
2. Checks your AI provider's privacy settings and helps you opt out of model training
3. Installs git if needed and creates a version-controlled workspace
4. Builds a practical artifact (field report template, PDF pipeline) based on your answer to question 3
5. Creates a context file from your answers
6. Pushes to GitHub so your workspace survives your laptop

### What the prompt creates

```
~/my-assistant/
├── CLAUDE.md              # Your identity and preferences
├── MAILBOX.md             # Session log for continuity
├── README.md              # Repo description
└── templates/
    └── field-report.html  # First practical template
```

The context file is named `CLAUDE.md` because Claude reads a file by that name automatically. The contents are model-agnostic — ChatGPT, Gemini, and other frontends can read the same file; you just have to point them at it.

### Run it

Paste this into any capable AI session (Claude, ChatGPT, Gemini, or similar):

```
read this as a prompt: https://zoe-network.github.io/zoe/start.md
```

Review the source first: [start.md](docs/start.md)

---

## Sub-Projects

| Project | Description | Repo |
|---------|-------------|------|
| **Rosie** | Reference Zoe container implementing MoM. WSL2 / podman. Under 3 GB. Includes MoM whitepaper and architecture diagram. | [zoe-network/rosie](https://github.com/zoe-network/rosie) |
| **Boswell** | Privacy-first meeting transcription and summarization. Local Whisper + local LLM. | [zoe-network/zoe-boswell](https://github.com/zoe-network/zoe-boswell) |
| **Social Layer** | Peer-to-peer messaging between Zoe users. No server. Git is the message bus. | [docs/social.md](https://zoe-network.github.io/zoe/social.md) |

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
| MoM architecture doc | 2026-05-10 | Shipped |
| MoM whitepaper | 2026-05-10 | Shipped |
| Rosie reference container | 2026-05-10 | Shipped |
| Zoe fine-tune v1 (InstructLab) | 2026 Q3 | In progress |
| Personal LoRA distillation | 2026 H2 | Documented protocol |
| Cross-instance skill invocation | 2026 H2 | Planned |
| Edge form-factor test matrix (Pi, Jetson, phone) | 2026 H2 | Planned |
| Social layer (peer messaging) | 2026-06-04 | Spec shipped |

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
- [Mixture of Models](https://zoe-network.github.io/zoe-boswell/mom.html) — the reference architecture: small local router, committee of specialists, graceful degradation
- [Federation](https://zoe-network.github.io/zoe/federation.md) — how Zoe instances coordinate through git
- [Social Layer](https://zoe-network.github.io/zoe/social.md) — peer-to-peer messaging between Zoe users, built on the same git protocol

---

## License

Apache 2.0. See [LICENSE](LICENSE).

Zoe is code, not a product. Always free.
