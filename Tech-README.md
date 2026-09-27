# 🔍 LLM Lens

> An agentic auditor that reveals how LLMs and AI agents perceive your website, and generates the fixes.

*"See your web the way LLMs do."*

LLM Lens audits a site's AI-readiness, meaning the files and markup that decide whether an agent can discover, understand and act on a page (`llms.txt`, `robots.txt`, `meta tags`, `JSON-LD` structured data, `WebMCP` manifests). It then generates the missing or broken artifacts, validates them against a quality gate, and opens a pull request. Every consequential decision passes through a human checkpoint.

**Target audience:** developers, technical founders, and AI engineers who want their projects discoverable and usable by LLMs and AI agents.

---

## How a session flows

```
URL ──► parallel audit workers ──► report + site profile ──► profile validated
    ──► pick an artifact ──► generate ──► validate ──► approve / re-optimize
    ──► export: download or GitHub PR
```

A directed graph with five `interrupt()` gates, not an open-ended chatbot loop. Questions can be asked at any gate and answered in place; asking never advances the session, and every loop (profile repair, artifact repair, re-optimization, chat turns) is capped.

---

## Supervisor multi-agent architecture

One LangGraph graph, the supervisor (`LLMLensAgent`), owns the itinerary and the checkpointed state (`LLMLensState`, per `thread_id`). Four specialised subagents do the work and never call each other: a supervisor node reads what a subagent needs from the state, calls it, and writes what comes back.

- **ReportAgent** consolidates the worker findings into a structured `AuditReport`, a human-readable narrative, and the `SiteProfile` everything downstream is generated from.
- **OptimizerAgent** generates the selected artifact, screening user feedback against the artifact's spec *before* spending a generation call.
- **ValidatorAgent** is the quality gate: four parallel check families (deterministic parsers, an LLM-as-judge binary checklist, regeneration diff, preservation diff) merged into feedback that drives a one-shot repair loop.
- **ChatAgent** answers questions inline at any gate, grounded in a Qdrant domain-knowledge index plus the audit state, with RAGAS-style faithfulness and context-precision checks on its own retrieval loop.

Around them:

- **Parallel audit workers.** `fetch_node` runs two `asyncio` layers: raw fetch, Playwright render, discoverability and SEO workers start together; the WebMCP worker follows the render and cross-references what the page *declares* (`toolname`-annotated forms) against what a real browser session *registers at runtime*.
- **Human-in-the-loop by construction.** Five gates, each exposing a fixed action menu; the model maps free text onto that menu rather than choosing the next step itself. Details: [documents/supervisor.md](documents/supervisor.md).
- **Bounded autonomy.** 1 validator repair per artifact, 1 profile repair, 1 user re-optimization per artifact, 25 chat turns per session. A known procedure with a human at each decision, traded deliberately against open-ended flexibility.
- **Context window management.** Real question/answer turns are marked as they are written and rolled into a running summary past a token budget, so the full `messages` log never reaches a prompt wholesale.
- **Multi-provider model layer.** Per-role model configs (supervisor/chat/report on Groq, optimizer on Codestral, judge on a small Ministral) with lazy cross-model fallback chains that only trigger on retryable failures.

See [documents/architecture.md](documents/architecture.md) for the full anatomy, and one document per subagent under [documents/](documents/).

## Quality: evals, LLM-as-judge, and CI gating

- **The ValidatorAgent is an LLM-as-judge pipeline.** Each target (site profile + the three LLM-driven artifacts) registers a `ValidationSpec`: deterministic parser checks, a binary judge checklist answered in a single structured-output call, a semantic-diff check on regenerations (did the pass do *only* what was asked), and a preservation check against the file the site already serves. Only ERROR-severity failures become repair feedback; warnings about known pipeline limits report but never loop.
- **The same checks back the eval suite.** `tests/evals/` (~35 behavioural and graph-flow tests) and `tests/judge/` (~30 tests asserting the judge's verdicts on labelled cases), written once and run in two places: at runtime they generate feedback, in evals they catch prompt regressions.
- **Prompt versioning via LangSmith Hub.** Every prompt (generation, judge checklists, intent classification, RAGAS graders) is a versioned hub artifact resolved by name, and each call records the prompt's commit hash in the trace. A prompt edit is a new version with an eval history, not an in-place change.
- **RAGAS-style checks run in production, not just in evals.** The ChatAgent grades its own retrieval (context precision, min 0.5, which gates a one-time retry with a wider net) and its own answer (faithfulness, min 0.7, which gates one regeneration). Both fail open: a broken check must never cost the user their answer.
- **CI gating via GitHub Actions.** Unit tests run on every push; the two LLM suites run on demand (`workflow_dispatch`, since they cost tokens) with an enforced **80% pass-rate threshold** that publishes a `evals/pass-rate` / `judge/pass-rate` commit status and fails the job below it. A push also marks both suites *pending* so a commit can't look green before they've graded it. Current suite pass rate: 85%+.

---

## Prompt-injection defense

The pipeline ingests third-party content (page HTML, the `robots.txt` and `llms.txt` a site already serves), none of it written by us. `src/guards/` scans each served artifact for content written to instruct AI readers and classifies it `clean` / `suspicious` / `hostile`:

- Suspicious content enters prompts *fenced*, marked as audited content rather than direction.
- Hostile content is withheld entirely; the report and the ChatAgent carry the verdict and the evidence, never the bytes.
- The validator runs the same scan over *generated* artifacts, so a poisoned input can't flow through into a pull request.

---

## Documentation

| Document | Covers |
|---|---|
| [documents/architecture.md](documents/architecture.md) | The supervisor/subagent anatomy: gates vs nodes vs subagents, the full session graph, all loops and their caps |
| [documents/supervisor.md](documents/supervisor.md) | How each gate classifies a user turn and where the router sends the session; the routing table for every gate |
| [documents/report-agent.md](documents/report-agent.md) | ReportAgent internals: consolidate → humanize ∥ extract_profile, token budgets, the profile repair loop |
| [documents/optimizer-agent.md](documents/optimizer-agent.md) | OptimizerAgent internals: the two artifact families, screening, overrides, feedback slots |
| [documents/validator-agent.md](documents/validator-agent.md) | ValidatorAgent internals: `ValidationSpec`, the four check families, severities, feedback synthesis |
| [documents/chat-agent.md](documents/chat-agent.md) | ChatAgent internals: question classification, state slicing, the context window, hostile artifacts |

---

## Tech stack

| Layer | Technology |
|---|---|
| Agent framework | LangGraph + LangChain (compiled subgraphs, `interrupt()`/`Command(resume=...)`, `MemorySaver` checkpoints) |
| Models | Per-role configs: Groq (`gpt-oss-120b`/`20b`) for supervisor, report & chat; Mistral (`codestral`) for generation; `ministral-8b` as judge, all via `init_chat_model` + lazy fallback chains |
| Structured output | Pydantic schemas bound via `with_structured_output` |
| Retrieval | Qdrant (`llm-lens-domain-knowledge`) + `mistral-embed` embeddings, RAGAS-style runtime grading |
| Prompt management | LangSmith Hub: versioned prompts, commit-hash tracing |
| Evals | `pytest` + `pytest-asyncio` + `respx`; LLM suites behind an opt-in `eval` marker; pass-rate gate script (`scripts/check_evals_pass_rate.py`) |
| Auditing | Playwright (render + runtime WebMCP discovery), PageSpeed Insights API, `httpx` + `bs4` |
| CI/CD | GitHub Actions: unit tests on push; eval & judge suites gated at 80% pass rate |
| Frontend | Reflex (Python → compiled React) chat UI with artifacts panel and live agent-activity log |
| Export | GitHub REST API (fork → branch → commit → PR) + local download |
| Tooling | `uv`, Python ≥ 3.11 |
