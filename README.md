<h1 align="center">Retold</h1>

<p align="center"><strong>Long-term memory for AI agents, with evidence, isolation, and permission to return nothing.</strong></p>

<p align="center">
  <a href="https://adimyth.in/retold/"><img alt="Documentation" src="https://img.shields.io/badge/docs-adimyth.in%2Fretold-black"></a>
  <a href="https://github.com/adimyth/retold/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/adimyth/retold/actions/workflows/ci.yml/badge.svg"></a>
  <img alt="Python 3.12 | 3.13" src="https://img.shields.io/badge/python-3.12%20%7C%203.13-blue">
  <a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-green"></a>
</p>

Retold gives Python agents long-term memory that would rather return nothing than inject a vaguely related fact. Facts, decisions, and dated experiences live in one SQLite file, each with a verbatim source quote and isolated by agent, user, project, or organization. Search explains why it returned a record or nothing.

Storage, embeddings, retrieval, and evidence checks run locally. Background transcript extraction and the optional utility-aware path use a configured Anthropic or OpenAI model. An agent gets five tools, `memory_search`, `memory_get`, `memory_write`, `memory_revise`, and `memory_forget`, which the Deep Agents and CrewAI adapters register.

## Architecture

Retold separates who requests a search from how retrieved records enter an answer.

| Mode | Behavior | Status |
| --- | --- | --- |
| `tool_only` | The agent receives the five memory tools and calls them when its task needs memory. | Default |
| `utility_aware` | The host drafts an answer, retrieves only for a specific missing fact, and admits records only when they would change the draft. | Opt-in |

The utility-aware path needs host-supplied model clients and `supported_bundle()` from `retold.policy.reference`. A store serves the bundle only after recording a passing fitness result. Otherwise, the path runs in shadow mode. Planner, provider, judge, timeout, and latency-budget failures all serve the original draft and record the reason.

![Utility-aware memory path: draft first, retrieve only for specific missing context, and revise only with admitted records](docs/assets/utility-aware-path.svg)

### How the decision works

1. **Receive the user turn.** The host receives the user query along with public context, such as the current conversation and system instructions.
2. **Draft an answer without conditional memory.** The model produces a usable first draft before Retold retrieves any stored records. This draft is the safe fallback.
3. **Check whether stored context is needed.** In parallel with drafting, the planner checks whether the answer may require a previous decision, user preference, project fact, or other specific memory. It sees only the kinds of memory available, not the records themselves.
4. **Keep the original draft when memory is unnecessary.** Ordinary questions stop here. Retold performs no search, adds no conditional memory to the model context, and introduces no retrieval latency beyond the planner running alongside the draft.
5. **Search for the missing context.** When memory may help, Retold searches specifically for the missing information. Before ranking candidates, it filters records by the caller's grants, scope, lifecycle state, type, and time constraints.
6. **Judge whether the retrieved context would change the draft.** Finding a related record is not enough. The admission judge compares each candidate with the existing draft and asks whether it would materially correct, complete, or personalize the answer.
7. **Keep the original draft when nothing is useful.** Retrieved but unnecessary records are rejected and never reach the answering model.
8. **Revise once when useful context exists.** The model receives only the admitted records and regenerates the answer once. Rejected candidates remain excluded.
9. **Serve the answer.** The final result is either the original memory-free draft or one revision informed by useful, authorized memory.


## Quick start

Python 3.12 or 3.13 is required. Install from GitHub Releases:

```bash
python -m pip install "retold[local-models] @ https://github.com/adimyth/retold/releases/download/v1.2.1/retold-1.2.1-py3-none-any.whl"
```

Store one memory and search for it:

```python
from retold import Retold

with Retold.open("memory.sqlite", profile="lite") as retold:
    with retold.session(user_id="aditya") as memory:
        memory.remember(
            "I prefer concise answers.",
            evidence="I prefer concise answers.",
            attribute="answer_style",
        )
        result = memory.search("What kind of answers do I prefer?")
        print(result.text)
```

It prints the stored claim and why it passed retrieval:

```text
Recalled 1 memory for "What kind of answers do I prefer?".

[01a08...] semantic · confirmed · user_statement · event 2026-09-10 · scope agent:assistant/aditya
I prefer concise answers.
matched: dense 0.53 (rank 1), lexical 2/3 (rank 1); fused rank 1; passed dense 0.53 ≥ 0.32 (semantic)
```

No API key is needed. The first run downloads about 640 MB of local models. [`examples/`](https://github.com/adimyth/retold/tree/main/examples) has the utility-aware quick start and complete Deep Agents, CrewAI, and OpenAI integrations.

## How memory works

### What kinds of memory Retold writes?

#### Conversation Memory

The current conversation, held in the model's context window. The app and the model handle this, not Retold.

#### Long-term memory

Retold keeps four kinds of memory in one store.

| Kind | What it holds | Example |
| --- | --- | --- |
| Facts and preferences | Things that stay true until they change | "The user prefers short answers." |
| Events and decisions | Things that happened, with a date | "On March 2 the team moved the launch to March 10." |
| Session summaries | A short account of one conversation, one per session | "Planned the launch. The user asked for shorter answers." |
| Procedures | How to do a task, kept in versions | "When changing the embedding model, re-embed every record." |

### When Retold writes memory?

Retold writes memory at two moments: when the agent chooses to save something during the conversation, and once after the conversation ends.

**Inline Processing:** During the conversation, the agent saves on purpose. The agent has a `memory_write` tool. When the user says "Keep answers short," the agent can save it right away.

**Post Processing:** After the conversation, a background job saves anything else worth keeping. A separate model rereads the whole transcript and picks out facts, decisions and a short summary of the session.

### What a memory looks like

Each fact, event or procedure is a small, single statement that carries the user's exact words as proof. Example:

| Part | Example |
| --- | --- |
| Statement | The user prefers short answers. |
| Proof | "Keep answers short." (quoted from the transcript) |
| Topic | this user + `answer_style` |
| Belongs to | this user, readable only by agents granted access |
| Source | the user said it, so it is trusted |

Three rules decide what gets in:

1. **No proof, no record.** If the quote can't be found in the transcript, an extracted item is thrown out, and an agent's save is downgraded to a guess.
2. **A second model checks every extracted item** before it is saved. It can accept it, reject it, or narrow it.
3. **Who said it sets the trust level.** Something the user said is confirmed. Something the agent concluded is a guess, and it expires after 30 days unless it comes up again.

Retold doesn't use domain-specific categories. Proof and review are what keep noise out.

### How Retold reads memory back

Memory reaches the model in up to three ways. Only the first is on by default, so Retold doesn't add memory to every turn unless you turn that on.

1. **The agent searches (default).** When the user asks "When's the launch again?", the agent calls its `memory_search` tool and gets back the March 10 record. A search took about 25 ms on 1,000 records and 75 ms on 50,000, measured on M4 Pro.
2. **A short profile at the start (optional).** A few stable preferences, such as answer style, language or accessibility needs, are loaded once when the conversation begins. The profile holds at most 8 items and stays fixed for the whole conversation.
3. **Draft first, then check (optional).** The model writes a draft answer without memory. Meanwhile a planner asks: "Is there something about this user that would change this answer?" If not, the draft is sent. If so, Retold searches for that missing fact, a judge checks whether any result really changes the draft, and only then does the model rewrite once.

### How memory stays bounded?

Retold limits growth mostly by updating records instead of adding new ones, and by letting guesses expire.

1. **Repeats strengthen instead of duplicating.** If a new item says nearly the same thing as an existing record, Retold strengthens the existing record and adds nothing.
2. **New facts replace old ones.** Each fact has a topic, such as "this user + answer style." When the user later says "Actually, I want detailed answers now," that record replaces "prefers short answers," and search stops returning the old one.
3. **Things expire.** See the table below.
4. **Reading back is capped, even as the store grows.** A search returns at most 8 records. The starting profile holds at most 8 items. Extraction keeps at most 20 items per conversation.

| What | How long it lasts |
| --- | --- |
| An agent's guess | 30 days, unless it comes up again |
| A session summary | 180 days |
| Something the user said | Until it is replaced or deleted |

Users and agents can also delete a memory with `memory_forget`.

### At a glance

| Question | How Retold answers it |
| --- | --- |
| What does it store? | Facts and preferences, dated events, session summaries and procedures. A few stable preferences can also load as a profile at the start of a conversation. |
| When does it write? | The agent saves memories on purpose during the conversation. After it ends, a background job saves anything else worth keeping. |
| What gets in? | Each memory is a single statement backed by the user's own words. A second model checks what the background job finds before it is saved. |
| How does it read back? | The agent searches when it needs a memory. Two optional ways add more: a profile loaded at the start, and a draft-first check that adds memory only when it would change the answer. |
| Where does the work run? | Searches and saves run inside the app while the user waits. The background job runs after the conversation ends. |
| How does it stay small? | New facts replace old ones instead of piling up, and unconfirmed guesses expire. Each search returns at most 8 memories. Related memories are not merged yet. |

## Inspiration

Retold's utility-aware path builds on two papers that reject retrieving by similarity and injecting the hits:

- [RUMS](https://arxiv.org/abs/2604.14473), *Response-Aware User Memory Selection for LLM Personalization* (ICML 2026), selects the stored memories that most reduce the model's uncertainty about its next response.
- [TRACE-Memory](https://arxiv.org/abs/2608.08446), *Public-Conditioned Retrieval and Utility-Aware Evidence Admission for Personalized Generation* (2026), plans the user-specific information a question is missing, retrieves for those gaps, and admits evidence only if it improves the answer beyond the public context.

Retold follows TRACE-Memory's two-stage shape with a prompted planner and a draft-relative judge, and adds an ambient-versus-conditional split, a content-free planner inventory, shadow mode, kill switches, and decision logs. [What is distinctive about this design](docs/design-contributions.md) compares the three.
