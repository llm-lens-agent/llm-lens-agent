# Anatomy of LLM Lens

## Supervisor and Subagents Architecture

LLM Lens is a **supervisor–subagent architecture**. One LangGraph graph, the supervisor (`LLMLensAgent`), owns the flow and owns the state. Four subagents do the specialised work: **ReportAgent**, **OptimizerAgent**, **ValidatorAgent** and **ChatAgent**.

The subagents never call each other. They cooperate through a single shared state, the supervisor's `LLMLensState`, checkpointed per `thread_id`: a supervisor node reads what a subagent needs from the state, calls it, and writes what comes back. The next subagent's node picks it up from there.

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 55, "rankSpacing": 180, "curve": "basis", "padding": 16}}}%%
flowchart LR
    state[("Supervisor<br/>LLMLensState<br/>checkpointed per thread_id")]

    RA["ReportAgent"]
    OA["OptimizerAgent"]
    VA["ValidatorAgent"]
    CA["ChatAgent"]

    state -->|"worker findings"| RA
    RA -.->|"report · site_profile"| state
    state -->|"site_profile · feedback"| OA
    OA -.->|"generated artifact"| state
    state -->|"the profile, or the artifact"| VA
    VA -.->|"findings to repair"| state
    state -->|"messages + the rest"| CA
    CA -.->|"answer · summary_update"| state

    classDef sup fill:#F1F3F5,stroke:#6B7280,stroke-width:2px,color:#14181D
    classDef report fill:#E4F4E8,stroke:#1B7F42,stroke-width:2px,color:#0F2A18
    classDef optimizer fill:#E5EEFB,stroke:#2160A8,stroke-width:2px,color:#0E2440
    classDef validator fill:#FCEDDC,stroke:#B85C10,stroke-width:2px,color:#3A2109
    classDef chat fill:#F0E8FA,stroke:#6B3FA0,stroke-width:2px,color:#26123F
    class state sup
    class RA report
    class OA optimizer
    class VA validator
    class CA chat
```


> **Legend**
> - 🟩 **ReportAgent** · 🟦 **OptimizerAgent** · 🟧 **ValidatorAgent** · 🟪 **ChatAgent** · ⬜ the supervisor and its state
> - **Arrows**, ➡️ solid is what the calling node passes in · ⇢ dotted is what the subagent returns, which the *node* then writes

That last distinction is how one subagent's output becomes another's input without either knowing the other exists:

| Subagent | Reads (via its node) | Returns (the node writes it) | Picked up by |
|---|---|---|---|
| **ReportAgent** | worker findings, `site_profile_feedback` | `report`, `humanized_summary`, `site_profile` | ValidatorAgent, OptimizerAgent, ChatAgent |
| **ValidatorAgent** | `site_profile`, or the generated artifact | `site_profile_feedback` / `validation_feedback`, `validation_results` | ReportAgent or OptimizerAgent |
| **OptimizerAgent** | `site_profile`, findings, `user_feedback`, `validation_feedback` | `generated_artifacts` | ValidatorAgent, ChatAgent, export node |
| **ChatAgent** | `messages` and the rest of the state | the answer and a `summary_update` | the gate that called it |

---

## Why it stops: ReAct agent versus supervisor agent

The default shape for an agent today is the **ReAct loop**: the model gets a goal and a set of tools, and on every iteration it decides which tool to call next, until it decides it is done. Control flow *is* the model's output. The human speaks at the start and reads at the end.

LLM Lens is a **supervisor agent**: one graph that owns the itinerary, four subagents that do the specialised work, and `interrupt()` at five points where the state is persisted and a person decides.

This is not a smaller kind of agent. The model still drives, reading intent, writing the report, extracting the site profile, generating each artifact, grading it and answering questions about it. What it does not do is choose which phase comes next. Autonomy sits *inside* each phase rather than in the routing between them.

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 40, "rankSpacing": 50, "curve": "basis"}}}%%
flowchart LR
    subgraph AUTO["ReAct agent · the model picks every step"]
        direction TB
        goal>"user: goal"] --> l1["LLM: which tool?"]
        l1 --> t1["tool call"]
        t1 --> l2["LLM: which tool?"]
        l2 --> t2["tool call"]
        t2 --> more["...until the model<br/>decides it is done"]
        more --> answer(["final answer"])
    end

    subgraph SUP["LLM Lens · supervisor agent"]
        direction TB
        u1>"User input"] --> g1(["gate: URL?"])
        g1 --> w1["audit + report"]
        u2>"User input"] --> g2(["gate: which artifact?"])
        w1 --> g2
        g2 --> w2["generate + validate"]
        u3>"User input"] --> g3(["gate: approve it?"])
        w2 --> g3
        u4>"User input"] --> g4(["gate: export where?"])
        g3 --> g4
        g4 --> w3["open the PR"]
    end

    AUTO ~~~ SUP

    classDef gate fill:#F1F3F5,stroke:#6B7280,stroke-width:2px,color:#14181D
    classDef node fill:#FBE4EE,stroke:#C2185B,stroke-width:1.5px,color:#4A0E2A
    classDef userin fill:#FFFFFF,stroke:#F4701F,stroke-width:1.5px,color:#F4701F
    class g1,g2,g3,g4 gate
    class l1,t1,l2,t2,more,answer,w1,w2,w3 node
    class goal,u1,u2,u3,u4 userin

    style AUTO fill:#FBFCFD,stroke:#9AA5B1,stroke-width:1.5px,color:#14181D
    style SUP fill:#FBFCFD,stroke:#9AA5B1,stroke-width:1.5px,color:#14181D
```

> **Legend**
> - ⬜ oval **Gate** :  the graph calls `interrupt()`, persists the state and waits for a person, and the oval shape sets it apart from a machine step
> - 🩷 box **Machine step** :  runs to completion with nobody watching
> - 👤 **User input** :  where a person types and the graph resumes; `user: goal` on the ReAct side plays the same role

| | ReAct agent | LLM Lens · supervisor agent |
|---|---|---|
| **Who picks the next step** | The model, every iteration | The graph's edges. The model only maps the user's text onto the gate's fixed set of actions |
| **Where the human is** | At the start and the end | At every decision with consequences: which artifact, whether to approve it, where to export it |
| **Loops** | Open until the model stops or a budget runs out | Each one capped: 1 repair pass, 1 re-optimization per artifact, 25 chat turns |
| **When it goes wrong** | Drifts, repeats tools, acts on a misread goal | Stops and asks |
| **Cost and latency** | Hard to predict | Bounded per phase |
| **Fits** | Open-ended exploration | A known procedure whose output lands in someone else's repo |

What it gives up is flexibility: at a given gate you can ask questions or pick from the menu, but you cannot send the agent off to do something the graph has no edge for. For a tool that ends by opening a pull request, that is the trade we want.

---

## Three kinds of piece: gates, nodes, subagents

Every step of the supervisor graph ( everything drawn as a box in the diagrams above and below) is one of three kinds. Most confusion about the flow comes from lumping them together. They are different things, and **only one of the three can touch the state**.

| Kind | Count | What it is |
|---|---|---|
| **Gate** | 5 | Calls `interrupt()`. The graph stops there and the user types. It is a node, so it does write to the state. |
| **Node** | 6 | Runs without intervention, returns a state *update*, and hands the turn back to the router. Also a node: it writes. |
| **Subagent** | 4 | An object a node instantiates and calls. Returns domain data. **Never touches the state, the node handles the LLMLens State update** |

---

## A session, end to end

The whole session, with every loop the graph actually has.

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 70, "rankSpacing": 100, "curve": "basis", "padding": 12}}}%%
flowchart TD
    ui1>"User input"] --> initial
    ui2>"User input"] --> post_audit
    ui3>"User input"] --> post_validation

    initial(["initial<br/>asks for the URL"])
    fetch["fetch<br/>webpage info"]
    invoke_report["ReportAgent"]
    validate_profile["ValidatorAgent"]
    post_audit(["post_audit<br/>menu: which artifact?"])
    invoke_optimizer["OptimizerAgent"]
    invoke_validator["ValidatorAgent"]
    post_validation(["post_validation<br/>do you approve it?"])
    reoptimize(["reoptimize_feedback<br/>collects your correction"])
    export(["export<br/>asks for the repo URL"])
    pr["pr<br/>opens the pull request"]
    done(["END"])

    initial -->|URL received| fetch
    initial -.->|"question / error"| initial
    fetch --> invoke_report
    invoke_report --> validate_profile
    validate_profile -.->|findings| invoke_report
    validate_profile --> post_audit
    post_audit -.->|"question / blocked option"| post_audit
    post_audit -->|artifact selected| invoke_optimizer
    invoke_optimizer -->|generated| invoke_validator
    invoke_validator -.->|findings| invoke_optimizer
    invoke_validator --> post_validation
    post_validation -.->|"question"| post_validation
    post_validation -->|approved| post_audit
    invoke_optimizer -.->|refused| reoptimize
    reoptimize -->|"User requested changes"| invoke_optimizer
    post_validation -->|wants changes| reoptimize
    post_validation ~~~ ui4
    ui4>"User input"] --> reoptimize
    post_validation -->|done, export| export
    ui5>"User input: GH URL"] --> export
    export -.->|"question"| export
    export -->|repo URL| pr
    pr --> done

    classDef gate fill:#F1F3F5,stroke:#6B7280,stroke-width:2px,color:#14181D
    classDef node fill:#FBE4EE,stroke:#C2185B,stroke-width:1.5px,color:#4A0E2A
    classDef userin fill:#FFFFFF,stroke:#F4701F,stroke-width:1.5px,color:#F4701F
    classDef report fill:#E4F4E8,stroke:#1B7F42,stroke-width:2px,color:#0F2A18
    classDef optimizer fill:#E5EEFB,stroke:#2160A8,stroke-width:2px,color:#0E2440
    classDef validator fill:#FCEDDC,stroke:#B85C10,stroke-width:2px,color:#3A2109
    class initial,post_audit,post_validation,reoptimize,export gate
    class fetch,pr node
    class invoke_report report
    class invoke_optimizer optimizer
    class invoke_validator,validate_profile validator
    class ui1,ui2,ui3,ui4,ui5 userin
```

> **Legend**
> - ⬜ oval **Gate**, where the user is sitting and waiting; the oval shape marks the five `interrupt()` points
> - 🩷 box **Node**, a step with no subagent behind it (`fetch`, `pr`)
> - 🟩 **ReportAgent** · 🟦 **OptimizerAgent** · 🟧 **ValidatorAgent**, every other box is named for the subagent it calls; **ValidatorAgent** appears twice, once for the site profile (`validate_profile`) and once for the generated artifact (`invoke_validator`)
> - 👤 **User input**, what you typed, arriving at that gate
> - ➡️ solid is the happy path · ⇢ dotted is a loop back, and every one of them is capped

Four things worth reading off the diagram:

- **Every gate routes questions back to itself.** Asking a question never advances the session, and you come back out standing exactly where you were. That loop is capped at `MAX_CHAT_TURNS = 25` per session, counted globally rather than per gate, because it is the same loop wherever the user happens to be standing.
- **Both repair loops are one-shot** (`MAX_PROFILE_REGENERATIONS` and `MAX_ARTIFACT_REGENERATIONS`, both `1`). The repair pass regenerates from an identical `SiteProfile`, so whatever survived the first correction is unlikely to fall to a third. When the cap is hit, the remaining findings are surfaced and the session moves on.
- **The optimizer has an early exit.** If screening refuses the user's request as incompatible with the spec, nothing was regenerated, so routing on to the validator would grade the *previous* version again and report it as fine. The user goes back to rephrase instead.
- **`reoptimize_feedback` is reached two different ways, for two different reasons.** From `invoke_optimizer` (`refused`), it means screening rejected the request before generating anything, and you are asked to rephrase. From `post_validation` (`wants changes`), it means the artifact passed validation and you chose not to approve it, and you are asked what to change. Same node, same next step (back to `invoke_optimizer`), but a different artifact version behind each one.

Export is reachable only from `post_validation`, not from the artifact menu: you export after approving something, not instead of generating it.

## Inside each subagent

Each one has its own compiled graph, invisible from the supervisor. What comes out of `run()` is domain data that the calling node decides how to use.

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 45, "rankSpacing": 70, "curve": "basis"}}}%%
flowchart TB
    subgraph RA["ReportAgent → report · humanized_summary · site_profile"]
        direction LR
        rin@{ shape: div-rect, label: "IN<br/>url · worker findings · profile_feedback" }
        rin --> consolidate["consolidate"]
        consolidate --> humanize["humanize"]
        consolidate --> extract["extract_profile"]
    end

    subgraph OA["OptimizerAgent → (generated, refinement_status, screening)"]
        direction LR
        oin@{ shape: div-rect, label: "IN<br/>site_profile · artifact_key · existing_artifact · feedback" }
        oin --> load["load_knowledge"]
        load --> screen["screen_feedback"]
        screen --> generate["generate"]
        screen -.->|incompatible| oend(["END"])
    end

    subgraph VA["ValidatorAgent → ValidationReport"]
        direction LR
        vin@{ shape: div-rect, label: "IN<br/>site_profile or generated artifact · existing_artifact · feedback" }
        vin --> vstart(["START"])
        vstart --> det["deterministic"] --> synth["synthesize_feedback"]
        vstart --> judge["judge"] --> synth
        vstart --> regen["regeneration"] --> synth
        vstart --> pres["preservation"] --> synth
    end

    RA ~~~ OA ~~~ VA

    classDef report fill:#E4F4E8,stroke:#1B7F42,stroke-width:2px,color:#0F2A18
    classDef optimizer fill:#E5EEFB,stroke:#2160A8,stroke-width:2px,color:#0E2440
    classDef validator fill:#FCEDDC,stroke:#B85C10,stroke-width:2px,color:#3A2109
    classDef terminal fill:#F1F3F5,stroke:#6B7280,color:#14181D
    classDef input fill:#FFFFFF,stroke:#9AA5B1,stroke-width:1.5px,color:#4B5563
    class consolidate,humanize,extract report
    class load,screen,generate optimizer
    class det,judge,regen,pres,synth validator
    class vstart,oend terminal
    class rin,oin,vin input

    style RA fill:#FBFCFD,stroke:#1B7F42,stroke-width:1.5px,color:#0F2A18
    style OA fill:#FBFCFD,stroke:#2160A8,stroke-width:1.5px,color:#0E2440
    style VA fill:#FBFCFD,stroke:#B85C10,stroke-width:1.5px,color:#3A2109
```

> **Legend**
> - 🟩 **ReportAgent** · 🟦 **OptimizerAgent** · 🟧 **ValidatorAgent**, every box here is a node of that subagent's *own* compiled graph
> - ⬜ **IN** (divided-process shape), the arguments the calling node passes to `.run()`; **START**/**END** belong to the subgraph, not to the supervisor
>
> Nothing in this picture can write to `LLMLensState`.

**ReportAgent** consolidates the findings and then, in parallel, writes the narrative and extracts the site profile. On a validator re-run it skips `humanize`: the narrative came from findings that have not changed. Its own document: [report-agent.md](./report-agent.md).

**OptimizerAgent** screens *before* generating. A request that clashes with the artifact's specification is refused and costs no more than that one call, which is what the early exit is for. Its own document: [optimizer-agent.md](./optimizer-agent.md).

**ValidatorAgent** fans four independent checks out from the start and has one junction that waits for all four before writing the verdict: `deterministic` (spec rules), `judge` (LLM grading), `regeneration` (did the repair actually fix it), and `preservation` (did it silently drop what the user asked for). Only ERROR-severity checks become feedback that loops, because warnings about known pipeline limits, like sparse HTML, would otherwise loop to the cap every single time. Its own document: [validator-agent.md](./validator-agent.md).

**ChatAgent** is a single node with two model calls and a no-model selection in between. It is the only subagent that reads `messages`, and it has its own document: [chat-agent.md](./chat-agent.md).

How each gate classifies what you typed and where the router sends the session next has its own document too: [supervisor.md](./supervisor.md).

Two artifacts, `robots.txt` and the WebMCP manifest, are assembled deterministically from findings, so there is no model output to grade. They get a placeholder validation and pass regardless.
