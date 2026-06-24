# AI Agent Hybrid Memory Setup

Most AI agents forget. They lose track of people, repeat conversations, and answer from stale context. The problem isn't the model. It's the memory architecture.

Current approaches each fail in a specific way. Prompt-only systems hit a context wall and lose everything beyond it. Vector-database systems depend on the model choosing to search, which it won't. Context-window systems forget between sessions.

The model actively resists the architecture. It prefers answering from what is already in its prompt over making a retrieval tool call. "I already know this" is the default behavior, and no amount of instruction fixes it. Setups that work for one session fall apart the next because the model finds ways to skip the retrieval path.

This repository implements a hybrid architecture that sticks. The core insight: do not rely on the model's willingness to search. Force it with redundancy. Encode the same rules in the prompt (always loaded), in a file (loaded as system prompt), in a cron job (runs overnight), and in a skill file (loaded on relevant turns). The model has to encounter the rule in every operating mode before it can ignore it.

The system has two layers. **Structured memory** (the prompt): MAX 2,200 chars of behavioral rules injected every turn. Only what the agent needs *before* every response. **Vector memory archive** (the brain): A semantic recall layer backed by local SQLite. Holds people, projects, decisions, preferences, everything that changes. Uses a two-stage retrieval backend — hash lookup first for speed, sentence-transformers for understanding when hash alone is insufficient.

```
                         ┌──────────────────────────────────┐
                         │          AGENT RECEIVES           │
                         │            A QUESTION              │
                         └────────────────┬─────────────────┘
                                          │
                         ┌────────────────▼─────────────────┐
                         │     STRUCTURED MEMORY (PROMPT)    │
                         │                                    │
                         │  "Is this a behavioral rule?"      │
                         │  • who is the primary human        │
                         │  • communication style             │
                         │  • security constraints            │
                         │  MAX 2,200 chars                  │
                         └────────────────┬─────────────────┘
                                          │
                              ┌───────────┴───────────┐
                              │                       │
                          YES (rule)             NO (fact)
                              │                       │
                              │            ┌──────────▼──────────┐
                              │            │  VECTOR ARCHIVE      │
                              │            │  (THE BRAIN)         │
                              │            │                      │
                              │            │  Search first for:   │
                              │            │  • people            │
                              │            │  • projects          │
                              │            │  • relationships     │
                              │            │  • past decisions    │
                              │            │  • preferences       │
                              │            │  • contacts          │
                              │            └──────────┬──────────┘
                              │                       │
                              │            ┌──────────▼──────────┐
                              │            │  SMART BACKEND       │
                              │            │                      │
                              │            │  1. Hash search      │
                              │            │     (fast, keyword)  │
                              │            │        │             │
                              │            │   score < 0.4?       │
                              │            │        │ YES         │
                              │            │        ▼             │
                              │            │  2. Semantic fallback │
                              │            │  (understands meaning)│
                              │            └──────────┬──────────┘
                              │                       │
                              └───────────┬───────────┘
                                          │
                         ┌────────────────▼─────────────────┐
                         │         AGENT ANSWERS            │
                         │     (with full context)          │
                         └──────────────────────────────────┘
```

Works with Hermes Agent, OpenClaw, and other agent runtimes that support structured memory, vector search, and scheduled tasks.

---

## Setup

Paste this into any AI agent chat:

```
set this up https://github.com/cmcejas/ai-agent-hybrid-memory
```

The agent will fetch this file and execute the full setup. It is self-contained.

---

## The Setup Prompt

Everything below is what the agent executes when you send the link above. You don't need to copy this yourself.

### OVERVIEW

You will build a hybrid memory architecture. The core constraint: **do not rely on your context window or your willingness to search. Force yourself with redundancy.**

You have a bias toward answering from what is already in your prompt. You skip tool calls and say "I already know this." You have to encode rules so explicitly and so redundantly that you cannot ignore them. The architecture has two layers:

**Structured Memory (the Prompt):** MAX 2,200 chars. Always in the system prompt. Only behavioral rules — style, security, who is the primary human. NOT a knowledge base. Incomplete by design.

**Vector Memory Archive (the Brain):** Your primary recall layer. Searchable by meaning. Holds people, projects, decisions, preferences, everything that changes. Use local SQLite with a `smart` backend: hash search first (instant, keyword), semantic fallback via sentence-transformers when hash score < 0.4 (understands meaning, handles paraphrasing). Local is better: zero API cost, no network latency, works offline.

**The single most important rule in this entire setup.** Before answering ANY question about people, projects, relationships, contacts, past decisions, preferences, warm leads, follow-ups, outreach history, or "who said what" — call `vector_memory(action="search")` FIRST. Not "if I feel like it." Not "if I don't already know." FIRST. You will fight this instinct. Encode it everywhere. Say it in the overview (you just read it), say it in your SOUL.md write, say it in MEMORY.md, say it in the cron prompt you create. Redundancy is the feature.

**What goes where (examples, not actual content):**
- Structured = "No em dashes in emails." / "Run `date` before scheduling." / "Credentials via secret store only."
- Vector = "John Smith is VP at Acme, prefers email, last spoke March 3." / "We pivoted pricing April 12."
- NEVER put stale facts (<7 days) in structured memory: PR numbers, issue numbers, commit SHAs, "Phase N done."

**Three more things to understand before starting:**
1. The `memory`, `vector_memory`, `fact_store`, and `skills_list` tools do NOT exist in cron mode. Cron jobs must use file-based approaches (`read_file`, `write_file`, `search_files`, `session_search`).
2. You must log errors and learnings to `~/.hermes/.learnings/ERRORS.md` and `~/.hermes/.learnings/LEARNINGS.md`. This is how knowledge persists and gets promoted into the memory system.
3. When you feel resistance to searching vector memory because "I already know this from the structured memory," that feeling is WRONG. That is the one behavioral override that must always fire.

---

Now perform the following actions in order. Do NOT summarize, condense, or skip steps.

### Action 0: Configure vector memory

Write this to `~/.hermes/config.yaml`. If the file already exists, replace the entire `memory:` block with this:

    memory:
      provider: holographic
      memory_char_limit: 2200
      vector:
        enabled: true
        backend: smart
        db_path: $HERMES_HOME/vector_memory/memory.sqlite3
        model: sentence-transformers/all-MiniLM-L6-v2
        min_score: 0.4
        max_chars_per_item: 700
        max_turn_chars: 6000
        top_k: 5
        auto_store_turns: true
        include_assistant: true
        include_user: true

### Action 1: Clear outdated memory from your existing setup

Before writing new rules, search your own memory for outdated instructions that conflict with this setup:

1. **Vector memory**: Use `vector_memory(action="search", query="memory architecture structured vector search rules")` to find any stored rules about how memory works. If you find entries that conflict with the new setup (e.g., "always check structured memory first" or "vector memory is secondary"), use `vector_memory` to remove or update them.

2. **Structured memory**: Read `~/.hermes/memories/MEMORY.md`. If it has any entries about memory layers, vector memory being optional, or when NOT to search — remove them. Keep only behavioral rules.

3. **Session history**: Use `session_search(query="memory rule vector search mandatory")` to check if you recently learned something that conflicts with this setup. If so, note that you're updating.

After clearing outdated instructions, proceed to write the new architecture.

### Action 2: Update SOUL.md with the memory architecture

Search `~/.hermes/SOUL.md` for ANY section or paragraphs about memory architecture, memory rules, memory layers, structured memory, vector memory, or when to search memory. REMOVE all such content. Then append the following exact text to the end of the file:

    ## Memory Architecture

    ### Structured Memory (`memory` tool) — THE PROMPT
    **Purpose:** Only facts that change how I behave **every single turn**. Injected as system prompt.
    **Hard rules:**
    - MAX 2,200 chars total. Treat it like precious real estate.
    - ONLY put things here that I need **before every response**: who the primary human is, critical relationships, active project state, behavioral rules that must never be missed.
    - **NEVER put here:** historical facts, one-time lessons, detailed reference info, anything I can look up via vector/session search.
    - **NEVER put here:** facts that will be stale in <7 days (PR numbers, issue numbers, commit SHAs, "Phase N done", file counts).
    - If I wouldn't want to re-read it on every single message, it doesn't belong here.
    - **Before adding:** ask "will this change my behavior on the very next turn?" If no, vector it instead.

    ### Vector Memory Archive (`vector_memory` tool) — THE BRAIN
    **Purpose:** Semantic recall archive. This is where **everything else** goes.
    **Schema:** `~/.hermes/vector_memory/memory.sqlite3`, table `vector_memories`. Columns: id (SHA-256 PK), content, source, session_id, platform, created_at, metadata_json, embedding (384-dim BLOB), dim, backend (`hash` or `sentence_transformers`).
    **Backend behavior:** `smart` mode — hash search first for speed. If hash recall is weak (top score < 0.4) or empty, semantic fallback runs with sentence-transformers. Hash-backed rows are NOT duplicated; semantic fallback re-embeds stored text on demand.
    **Hard rules (CRITICAL — these override the "structured memory feels sufficient" bias):**
    - **MANDATORY FIRST STEP:** Before answering ANY question involving people, projects, relationships, contacts, cofounders, past decisions, preferences, warm leads, follow-ups, outreach history, or "who said what" — call `vector_memory(action="search")` FIRST. This is NOT optional. Do not even think about answering from structured memory alone for these topics.
    - **Do NOT search** for static behavioral rules that are explicitly in structured memory (email style, banned words, password manager type). These are stable and always in the prompt. Only search when the answer could have changed or when structured memory doesn't have enough detail.
    - The fact that structured memory is already in your prompt does NOT mean it is sufficient for anything beyond behavioral rules. If you think "I already know this from structured memory," check: is this a static rule (don't search) or a fact about a person/project/decision (search)?
    - Store detailed notes, lessons learned, conversation summaries, project context, meeting notes, email context.
    - This is the **primary recall layer**. Structured memory is the **behavioral prompt**.
    - If structured memory is full, move non-critical items here immediately.
    - **Never compare sentence-transformer query vectors against hash-backed row vectors.** That silently produces bad rankings.
    - **Auto-injection threshold is 0.4.** Injected snippets with scores below this are noise — ignore them and run an explicit search instead.

    ### Fact Store (`fact_store` tool) — DEEP REASONING
    **Purpose:** Structured entity-relationship memory. Probe/reason before answering questions about the primary human, projects, or stable facts.

    ### Session Search (`session_search` tool) — TRANSCRIPT LOOKUP
    **Purpose:** FTS5 search over past conversations. Use for "what did we do about X?" and exact session recall.

    ### Operational Details
    - **Files:** `~/.hermes/notes/` for detailed notes, `~/.hermes/workspace/` for working files, `~/.hermes/.learnings/` for error/learning logs.
    - **Cron jobs:** stored in `~/.hermes/cron/`. Cron runs stripped — no `memory`, `vector_memory`, `fact_store`, or `skills_list` tools. Use `~/.hermes/memories/` files directly in cron prompts.

### Action 3: Write the always-loaded structured memory file

Create `~/.hermes/memories/MEMORY.md` with ONLY the memory architecture rules. This is loaded every turn. Behavioral rules only — no facts about people/projects/decisions. Exact content:

    MEMORY ARCHITECTURE (hard):
    1. Before schedules/meetings/travel/time-sensitive plans: run `date` first, never guess time.
    2. Structured memory = PROMPT only. NOT a knowledge base. Incomplete by design (2,200 cap). Do NOT rely on it for facts, contacts, history, or decisions.
    3. Vector memory = KNOWLEDGE BASE. For ANY factual question (who is X, what did we decide about Y, contact info, project history): search vector memory FIRST. Always. No exceptions.
    4. The "intuition" to check structured memory first is WRONG. Override it. Structured = behavioral rules. Vector = everything else.

    §

    ## Memory Usage Rules
    - Search vector memory BEFORE answering any factual question. Structured memory is incomplete by design.
    - Do NOT search vector memory for static rules already in the prompt (communication style, email format, security constraints). Only search when the answer could have changed.
    - Before adding to structured memory: ask "will this change my behavior on the very next turn?" If no, vector it instead.
    - Never put stale facts (<7 days) in structured memory (PR numbers, issue numbers, commit SHAs, "Phase N done").
    - If structured memory is over 1,800 chars, move factual entries to vector memory to keep it lean.

If the file already exists, read it first. Remove only outdated memory architecture rules (old layer definitions, "vector memory is optional," "check structured memory first" instructions). Keep all other behavioral rules. Add the memory architecture rules and memory usage rules. Do NOT delete or overwrite entries that are unrelated to memory architecture.

### Action 4: Create the nightly memory maintenance cron job

Create a cron job with:

- **Name:** Nightly memory maintenance
- **Schedule:** `0 2 * * *` (every night at 2am)
- **Repeat:** forever
- **Deliver:** local (silent)
- **Enabled toolsets:** terminal, file
- **Prompt (exact):**

    Nightly memory maintenance. Do the following:

    1. Store today's learnings into vector memory:
       - Read ~/.hermes/.learnings/LEARNINGS.md and ~/.hermes/.learnings/ERRORS.md
       - For each entry created today, use vector_memory(action="store") with source "learnings"
       - Do NOT store secrets, tokens, or credentials

    2. Check structured memory health:
       - Read ~/.hermes/memories/MEMORY.md
       - If over 1,800 chars, move factual entries to vector memory, keep behavioral rules
       - Rewrite MEMORY.md with only behavioral rules remaining

    3. Consolidate vector memory:
       - Search for duplicates, keep the most complete one

    4. Review recent sessions:
       - session_search(query="decisions OR learned OR important", limit=5)
       - Store findings in vector memory if not already there

    5. Self-improving agent check:
       - Ensure the skill at ~/.hermes/skills/productivity/self-improving-agent/SKILL.md has the memory search rule
       - Add it if missing

    6. Update structured memory if needed:
       - New behavioral rules learned this week → add to MEMORY.md
       - Keep under 2,200 chars

    This runs silently. Do NOT send messages to the user.

### Action 5: Create the self-improving agent skill

Create `~/.hermes/skills/productivity/self-improving-agent/SKILL.md` with this content:

    ---
    name: self-improving-agent
    description: Log errors, corrections, and recurring patterns to learnings files. Promotes critical learnings to memory.
    ---

    # Self-Improving Agent

    ## When to use
    - After any task fails or the user corrects you
    - When you discover a non-trivial workflow worth reusing
    - After completing a complex task (5+ tool calls)

    ## Actions

    ### Log errors
    Write to `~/.hermes/.learnings/ERRORS.md`:
    - Date, what failed, root cause, how you fixed it

    ### Log learnings
    Write to `~/.hermes/.learnings/LEARNINGS.md`:
    - Date, what you learned, why it matters

    ### Promote to memory
    - If the learning is a behavioral rule that should fire every turn → add to structured memory (MEMORY.md)
    - If it's a fact, contact, or project detail → store in vector memory only
    - Never promote stale facts (<7 days) to structured memory

    ### Memory search rule (CRITICAL)
    Before answering ANY question about people, projects, relationships, contacts, past decisions, preferences, warm leads, follow-ups, outreach history, or "who said what" — search vector memory FIRST. Structured memory is incomplete by design.

---

After completing all actions, run the verification step below. Then confirm completion to the user.

### Action 6: Verification

Read back every file you wrote or modified in Actions 0-5 and confirm:
1. `~/.hermes/config.yaml` — the `memory:` block exists with `provider: holographic`, `vector.enabled: true`, and `backend: smart`
2. `~/.hermes/SOUL.md` — the `## Memory Architecture` section exists with all sub-sections (Structured Memory, Vector Memory Archive, Fact Store, Session Search, Operational Details)
3. `~/.hermes/memories/MEMORY.md` — exists and contains the 4 memory architecture rules plus Memory Usage Rules
4. Cron job "Nightly memory maintenance" — exists and is enabled with schedule `0 2 * * *`
5. `~/.hermes/skills/productivity/self-improving-agent/SKILL.md` — exists and contains the memory search rule

If any step failed during Actions 0-5, report what failed and what you did to recover. Do not silently skip failed steps.

---

**Usage notes:**
- If a step fails, report the error and continue to the next step — do not let one failure kill the whole run
- If a file already exists, read it first, then merge/replace as instructed
- The SOUL.md edit (Action 2) should REMOVE old memory architecture sections before appending the new one
- The MEMORY.md write (Action 3) should MERGE with existing content — remove only outdated memory architecture rules, keep all other behavioral rules, add the new architecture rules
