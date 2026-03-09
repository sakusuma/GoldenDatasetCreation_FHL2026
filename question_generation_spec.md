# Question Generation from Teams Channel Reply Chains — Agent Spec

## §1 Overview

This spec defines the behavior of an **LLM agent** (Claude Code or equivalent) whose job is to read a Teams Channel conversation JSON file, understand the overall context of the channel, and generate **60 questions total**:

- **40 positive (answerable) questions** — 20 standard grounded questions + 20 vector search paraphrase questions.
- **20 negative (unanswerable) questions** — realistic-sounding questions that are definitively NOT related to or answerable from the channel content.

The combined 60-question dataset is designed to measure **precision, recall, and fidelity** of a RAG system with threshold gating. Standard positive questions validate recall and answer quality; negative questions validate the system's ability to correctly reject out-of-scope queries. Vector search positive questions specifically stress-test the semantic retrieval layer by using paraphrased phrasing to verify synonym and concept matching.

### §1.1 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Executor | Claude Code (LLM agent) directly | No intermediate Python/Azure OpenAI pipeline — the agent reads this spec + input file and produces output |
| Output format | CSV | Flat, portable, easy to validate |
| Question grounding | Positive questions must be answerable from message content | No manufactured/hallucinated questions for positives |
| Negative questions | 20 real-sounding but completely unrelated questions | Enables precision/threshold measurement in RAG eval |
| Vector search positive questions | 20 paraphrase questions (`QuestionType=vector_search`) | Tests semantic retrieval layer — same facts as channel, expressed with synonym/concept rewrites |
| Validation | 4 mandatory passes | Catches ID errors, answer accuracy, likelihood miscategorization, negative question leakage, and vector search paraphrase quality |
| PII handling | Retain author names as-is in output | Names are needed for realistic questions; downstream consumers handle redaction |
| `QuestionType` column | `standard` or `vector_search` | Distinguishes baseline evaluation questions from questions designed to stress-test the vector search / semantic retrieval layer |

### §1.2 Inputs & Outputs

| Artifact | Format | Location |
|----------|--------|----------|
| Input file | JSON array of thread objects | `LoadedChannelDataDump/<filename>.json` |
| Output file | CSV (60 rows + header) | `LoadedChannelDataDump/<filename>_questions.csv` |
| This spec | Markdown | `LoadedChannelDataDump/question_generation_spec.md` |

---

## §2 Input Data Schema

The input is a JSON array where each element is a **thread object**:

```json
{
  "id": "1762804137000",
  "message": {
    "content": "<p>HTML content of root message</p>",
    "createdDateTime": "2025-11-10T19:48:57Z",
    "fromId": "8:orgid:e75a9fb6-b489-4ae9-93c0-e55d38f83df9",
    "authorName": "Deepak Modi"
  },
  "replies": {
    "messages": [
      {
        "id": "1762842847000",
        "content": "HTML or plain text reply content",
        "createdDateTime": "2025-11-11T06:34:07Z",
        "fromId": "8:orgid:a97db061-24ad-4d1f-9f71-89a78cd242d5",
        "replyToId": "1762804137000",
        "authorName": "Bhawnagarg"
      }
    ]
  }
}
```

### §2.1 Key Fields

| Field | Location | Description |
|-------|----------|-------------|
| `id` | Thread root | Unique thread identifier (timestamp-based string) |
| `message.content` | Thread root | HTML-formatted root message content |
| `message.createdDateTime` | Thread root | ISO 8601 timestamp |
| `message.authorName` | Thread root | Display name of thread starter |
| `replies.messages[]` | Thread | Array of reply messages (may be empty) |
| `replies.messages[].id` | Reply | Unique reply message identifier |
| `replies.messages[].replyToId` | Reply | Always equals the parent thread `id` |
| `replies.messages[].content` | Reply | HTML or plain-text reply content |

### §2.2 Content Considerations

- **HTML stripping**: Message `content` contains raw HTML (`<p>`, `<blockquote>`, `<a>`, `<table>`, `<span>` with @mentions). The agent must parse through HTML to understand the semantic content.
- **Quoted replies**: `<blockquote>` elements contain quoted text from prior messages — these provide conversation threading context.
- **@mentions**: `<span itemscope="" itemid="N">Name</span>` indicates a user mention.
- **Embedded data**: Some messages contain `<table>` elements with structured data (e.g., Kusto query results, error counts).
- **Links**: `<a href="...">` elements contain URLs to PRs, ADO work items, Kusto queries, documents.

---

## §3 Question Categories (Positive Questions Only)

The agent must generate positive questions across exactly these **11 categories**:

| # | Category ID | Description | Typical Question Pattern |
|---|-------------|-------------|--------------------------|
| 1 | `information_gathering` | Questions seeking specific factual details from messages | "What error was reported for the meeting_scheduler skill?" |
| 2 | `summary_compilation` | Questions requesting a summary across multiple messages/threads | "Summarize the discussion about enableToolsStrictMode" |
| 3 | `decision_and_action_history` | Questions about decisions made, actions taken, or action items assigned | "What decision was made about disabling the flag in prod?" |
| 4 | `risk_and_issue_identification` | Questions about risks, bugs, regressions, or problems surfaced | "What risks were identified with R0 users testing prod bots?" |
| 5 | `status_and_progress` | Questions about current state, deployment status, or progress of work items | "What is the deployment status of the manage_tasks fix?" |
| 6 | `information_retrieval` | Questions asking to locate specific artifacts (links, PRs, docs, queries) | "Where is the PR for disabling Task Update v2 from R0?" |
| 7 | `knowledge_and_contextual_help` | Questions requiring understanding of technical concepts discussed | "What does the enableToolsStrictMode flag control?" |
| 8 | `progress_tracking` | Questions about timeline, milestones, or progress of specific features | "When did the LLM Errors issue first appear and when was the fix merged?" |
| 9 | `high_level_summary` | Questions requesting broad overviews of a topic, period, or the entire channel | "What are the main topics discussed in this channel over the last month?" |
| 10 | `task_tracking` | Questions about specific tasks, work items, bug assignments, or ownership | "Who is assigned to investigate Bug 4941593?" |
| 11 | `issue_compilation` | Questions asking to compile/aggregate all issues, errors, or problems discussed | "List all the LLM schema validation errors mentioned in the channel" |

### §3.1 Category Distribution Target

Aim for a roughly even distribution across categories (~2 questions each), but allow natural variance based on channel content. Categories with rich supporting data may have more questions. **No category should have fewer than 1 or more than 4 questions.**

> **Chunk constraint:** `summary_compilation` and `status_and_progress` questions **must always use `ChunkCategory=multi`**. Do not generate single-chunk questions for these two categories.

---

## §4 Chunk Categories (Positive Questions Only)

Each positive question must be classified as either **single-chunk** or **multi-chunk**:

| Chunk Category | Value | Definition | Criteria |
|----------------|-------|------------|----------|
| Single chunk | `single` | Answer is fully derivable from a **single reply chain** (one thread: root message + its replies) | `ParentMessageIds` contains exactly 1 thread ID. All `MessageIds` share the same `replyToId`. |
| Multi chunk | `multi` | Answer requires information from **2 or more distinct reply chains** (threads) | `ParentMessageIds` contains 2+ thread IDs. `MessageIds` span messages from different threads. |

### §4.1 Chunk Distribution Target

| Chunk Category | Target % |
|----------------|----------|
| `single` | 55–65% |
| `multi` | 35–45% |

---

## §5 Likelihood Classification (Positive Questions Only)

Each positive question is rated by how likely a real user of this channel would ask it:

| Likelihood | Value | Target % | Count (of 20) | Definition |
|------------|-------|----------|-----------------|------------|
| High | `High` | 70% | 14 | Common, natural questions any active channel member would ask |
| Medium | `Medium` | 28% | 5 | Reasonable but less frequent — requires deeper curiosity or specific context |
| Rare | `Rare` | 2% | 1 | Valid but unusual — edge-case questions a user *could* ask but rarely would |

### §5.1 Likelihood Guidelines

**High likelihood** questions:
- Ask about active bugs, errors, or regressions
- Request status of ongoing work items  
- Ask who is responsible for something
- Request a summary of a recent discussion
- Ask for specific links/PRs/work items mentioned

**Medium likelihood** questions:
- Require correlating information across multiple threads
- Ask about historical context or timelines
- Request comparison between different approaches discussed
- Ask about impact/scope of issues

**Rare likelihood** questions:
- Ask about obscure technical details only mentioned briefly
- Request aggregation across the entire channel history
- Ask about hypothetical scenarios based on discussed patterns
- Combine topics from unrelated threads

---

## §6 Output CSV Schema

The output is a **single UTF-8 CSV file** with a header row and **60 data rows** (40 positive + 20 negative).

| Column | Type | Description | Positive Example | Negative Example |
|--------|------|-------------|------------------|------------------|
| `IsAnswerable` | enum | `yes` for positive questions, `no` for negative questions | `yes` | `no` |
| `QuestionCategory` | enum | One of the 11 category IDs from §3 (positive) or empty (negative) | `risk_and_issue_identification` | *(empty)* |
| `ChunkCategory` | enum | `single` or `multi` (positive) or empty (negative) | `single` | *(empty)* |
| `Question` | string | The question text | `What was the root cause of the meeting_scheduler schema validation errors?` | `What was the outcome of the Q3 sales pipeline review meeting?` |
| `ExpectedAnswer` | string | Concise answer from referenced messages (positive) or empty (negative) | `The root cause was missing 'additionalProperties' in the schema...` | *(empty)* |
| `ParentMessageIds` | string | Pipe-delimited thread IDs (positive) or empty (negative) | `1762804137000` | *(empty)* |
| `MessageIds` | string | Pipe-delimited message IDs (positive) or empty (negative) | `1762805832000\|1762842734000` | *(empty)* |
| `Likelihood` | enum | `High`, `Medium`, or `Rare` (positive) or empty (negative) | `High` | *(empty)* |
| `RecencyCategory` | enum | `Recent50`, `OlderThanRecent50`, or `Mix` (positive) or empty (negative) — see §6.4 | `Recent50` | *(empty)* |
| `QuestionType` | enum | `standard` for baseline questions; `vector_search` for paraphrase/adversarial questions — see §14 | `standard` | `standard` |

### §6.4 RecencyCategory Classification (Positive Questions Only)

The `RecencyCategory` column classifies each positive question based on whether its referenced threads fall within the **50 most recent threads** in the input JSON. Thread recency is determined by sorting all top-level threads by `message.createdDateTime` in descending order and selecting the first 50.

| Value | Definition | Criteria |
|-------|------------|----------|
| `Recent50` | All referenced threads are among the 50 most recent | Every ID in `ParentMessageIds` belongs to the Recent50 set |
| `OlderThanRecent50` | All referenced threads are older than the 50 most recent | No ID in `ParentMessageIds` belongs to the Recent50 set |
| `Mix` | Referenced threads span both recent and older | At least one `ParentMessageId` is in the Recent50 set AND at least one is not |

#### RecencyCategory Distribution Target (of 20 positive questions)

| RecencyCategory | Target % | Count |
|-----------------|----------|-------|
| `Recent50` | 20% | 4 |
| `OlderThanRecent50` | 60% | 12 |
| `Mix` | 20% | 4 |

For **negative questions**, `RecencyCategory` is empty.

### §6.1 CSV Formatting Rules

- Delimiter: comma (`,`)
- Quoting: wrap any field containing commas, newlines, or double-quotes in double-quotes. Escape internal double-quotes by doubling (`""`)
- Encoding: UTF-8 with BOM
- Line endings: CRLF (Windows)
- No trailing comma on any row
- `ParentMessageIds` and `MessageIds` use pipe (`|`) as internal delimiter
- **ID fields must always end with a trailing pipe** — write `"1762804137000|"` for a single ID and `"1762804137000|1762805832000|"` for multiple IDs. This ensures the field value is never a bare integer, which prevents spreadsheet tools (e.g. Excel) from silently converting large IDs to scientific notation (e.g. `1.76251E+12`). Both fields must always be quoted and always end with `|`.
- Empty fields for negative questions: use empty string (two consecutive commas `,,` or `""`)

### §6.2 Row Ordering

All 40 positive questions first (rows 1–40), then all 20 negative questions (rows 41–60). Within the positive block: standard positives occupy rows 1–20, vector search positives occupy rows 21–40.

### §6.3 Sample Output Rows

```csv
IsAnswerable,QuestionCategory,ChunkCategory,Question,ExpectedAnswer,ParentMessageIds,MessageIds,Likelihood,RecencyCategory,QuestionType
yes,information_gathering,single,"What error message was being triggered by the meeting_scheduler skill?","The error was 'Invalid schema for function meeting_scheduler' caused by missing 'additionalProperties' field in the skill schema when 'enableToolsStrictMode' was set to true.","1762804137000|","1762804544000|1762805832000|1762842734000|",High,Recent50,standard
no,,,"What was the outcome of the Q3 sales pipeline review meeting?",,,,,,standard
no,,,"How do I configure SSO for the Salesforce integration in our CRM system?",,,,,,standard
yes,information_gathering,single,"What malformed tool definition issue caused failures in the scheduling component?","The error was 'Invalid schema for function meeting_scheduler' caused by missing 'additionalProperties' field in the skill schema when 'enableToolsStrictMode' was set to true.","1762804137000|","1762804544000|1762805832000|1762842734000|",High,Recent50,vector_search
```

---

## §7 Positive Question Generation Rules (Standard — `QuestionType=standard`)

> These rules apply to the **20 standard positive questions** (`QuestionType=standard`). For the 20 vector search positive questions, see §14.

### §7.1 Grounding Requirements

1. **Every positive question MUST be answerable from the message content.** The agent must not invent facts, speculate, or ask about topics not present in the data.
2. **ExpectedAnswer MUST be sourced exclusively from the referenced MessageIds.** Do not synthesize answers from general knowledge.
3. **MessageIds MUST be real IDs present in the input file.** Every ID in `MessageIds` must exist as either a thread `id` or a `replies.messages[].id` in the input JSON.
4. **ParentMessageIds MUST be real thread-level IDs.** Every ID in `ParentMessageIds` must appear as a top-level thread `id` in the input JSON.
5. **ParentMessageIds must be the parent threads of the referenced MessageIds.** For any message ID in `MessageIds`, its parent thread ID must appear in `ParentMessageIds`.
6. **ParentMessageIds and MessageIds serve different purposes.** `ParentMessageIds` identifies which thread(s) contain the answer (thread-level scope). `MessageIds` identifies the specific reply-level messages whose content directly grounds the answer. A thread root ID (`thread.id`) should appear in `MessageIds` **only** when the root message's own content is part of the answer — not as a default or fallback.
7. **ID fields must be written with a trailing pipe and quoted.** Write `"1762804137000|"` for a single ID and `"1762804137000|1762842734000|"` for multiple IDs. Never write a bare unquoted integer — spreadsheet tools silently convert large integers to scientific notation (e.g. `1.76251E+12`), which breaks all downstream ID lookups.
8. **MessageIds must reference the original source, not a quoting reply.** When a reply quotes an earlier message via `<blockquote>`, the `MessageIds` must point to the **original message** that first stated the information, not the quoting reply. The quoting reply may be included as an *additional* ID only if it adds new information beyond the quote.
9. **Preserve tense and conditionality exactly.** Do not convert future-tense, planned, or conditional statements into present-tense facts. If a source message says "target is Private Preview Jan 2026" or "we plan to go GA after…", the `ExpectedAnswer` must reflect that framing — never assert a planned milestone as a current accomplished fact.
10. **Named artifacts must be quoted character-for-character.** Bug titles, ADO work item titles, PR descriptions, feature names, error message strings, flag names, and API names must be copied verbatim from the source. Do not compress, rephrase, or summarise them — even minor rewording (e.g. "no runtime toggle" instead of "Provide an option to toggle … via runtime flights") constitutes a C1 failure.

### §7.2 Question Quality Requirements

1. **Specific, not generic.** Bad: "What bugs were discussed?" Good: "What was Bug 4941593 about and who created it?"
2. **Naturally phrased.** Questions should sound like a real team member typing in a search box or asking Copilot.
3. **Varied question styles.** Mix of who/what/when/where/why/how/list/summarize.
4. **No duplicate or near-duplicate questions.** Each question must target distinct information.
5. **Multi-chunk questions must genuinely require multiple threads.** Don't inflate chunk count artificially — if the answer lives in one thread, classify as `single`.
6. **`summary_compilation` and `status_and_progress` questions must always be classified as `multi` chunk.** These categories by nature require synthesising information across multiple threads. Never assign `ChunkCategory=single` to questions in these two categories.

### §7.3 HTML Content Handling

When reading message content to formulate questions and answers:
- Strip all HTML tags to extract plain text
- Resolve `<blockquote>` to understand quoted context
- Parse `<table>` elements to extract tabular data
- Treat `<a href="...">` text as link references (include URLs in answers when relevant)
- Resolve `<span itemscope>` elements as @mentions

---

## §8 Negative Question Generation Rules (Standard — `QuestionType=standard`)

> These rules apply to the **20 standard negative questions** (`QuestionType=standard`).

### §8.1 Core Principle

Every negative question must be a **realistic, natural-sounding question** that a real person might ask in some workplace context — but it must be **completely unrelated** to any topic, person, event, tool, system, or conversation in the input file. There should be **zero ambiguity** that the question has any match in the channel data.

### §8.2 Requirements

1. **No topical overlap.** The question must not touch any subject discussed in the channel — not the same projects, tools, people, teams, technologies, or events.
2. **No keyword overlap.** Avoid using distinctive terms, product names, skill names, feature flags, person names, tool names, or any specific nouns that appear in the input file. The agent must first inventory the key terms in the channel and actively avoid them.
3. **Realistic and specific.** Questions must sound like real workplace questions a person would actually ask — not nonsensical, absurd, or obviously fabricated. They should reference plausible (but unrelated) projects, people, systems, and timelines.
4. **Diverse topics.** Spread across a wide variety of unrelated domains — e.g., HR processes, marketing campaigns, hardware procurement, office facilities, finance/budgeting, unrelated product lines, customer support workflows, unrelated engineering teams, etc.
5. **Varied question styles.** Mix of who/what/when/where/why/how/list/summarize — similar natural phrasing to the positive questions.
6. **No duplicates.** Each negative question must be distinct.

### §8.3 Field Values for Negative Questions

| Field | Value |
|-------|-------|
| `IsAnswerable` | `no` |
| `QuestionCategory` | *(empty)* |
| `ChunkCategory` | *(empty)* |
| `Question` | The negative question text |
| `ExpectedAnswer` | *(empty)* |
| `ParentMessageIds` | *(empty)* |
| `MessageIds` | *(empty)* |
| `Likelihood` | *(empty)* |
| `RecencyCategory` | *(empty)* |

### §8.4 Examples of Good Negative Questions

> - "What was the final headcount approved for the Austin office expansion?"
> - "When is the next all-hands meeting for the Azure DevOps marketing team?"
> - "Who approved the vendor contract for the new cafeteria management system?"
> - "What are the updated guidelines for submitting expense reports in Concur?"
> - "How do I request a hardware refresh for my team's laptops?"
> - "What was the resolution of the customer escalation from Contoso about billing discrepancies?"
> - "When did the Dynamics 365 team migrate their CI/CD pipeline to GitHub Actions?"
> - "What is the PTO policy for contractors in the London office?"

### §8.5 Anti-Patterns (Bad Negative Questions)

| Bad Example | Why it's bad |
|-------------|-------------|
| "What is 2 + 2?" | Not a realistic workplace question |
| "Tell me a joke" | Not a realistic information-seeking question |
| "What LLM errors were reported last month?" | Overlaps with channel content (LLM errors are a major topic) |
| "Who is Deepak Modi?" | Uses an author name from the channel |
| "What is the status of the MCP integration?" | MCP is a core channel topic |
| "How does the orchestrator work?" | Orchestrator is discussed extensively in the channel |

---

## §9 Validation Protocol

The agent **MUST** execute exactly **4 sequential validation passes** after initial generation. Each pass reviews ALL 60 questions and corrects any errors found.

### §9.1 Pass 1 — ID Validation (Positive Questions)

For each positive row (`IsAnswerable=yes`), verify:

| Check | Rule | Fix if violated |
|-------|------|-----------------|
| V1.1 | Every ID in `ParentMessageIds` exists as a top-level thread `id` in the input JSON | Replace with correct thread ID or regenerate question |
| V1.2 | Every ID in `MessageIds` exists as either a thread `id` or `replies.messages[].id` in the input JSON | Replace with correct message ID or regenerate question |
| V1.3 | For each message ID in `MessageIds`, its parent thread appears in `ParentMessageIds` | Add missing parent thread ID to `ParentMessageIds` |
| V1.4 | `ParentMessageIds` has exactly 1 ID when `ChunkCategory` is `single` | Correct `ChunkCategory` to `multi` if >1, or reduce to 1 parent if answer truly comes from one thread |
| V1.5 | `ParentMessageIds` has 2+ IDs when `ChunkCategory` is `multi` | Correct `ChunkCategory` to `single` if only 1, or find additional evidence threads |
| V1.6 | No empty `ParentMessageIds` or `MessageIds` | Regenerate question |
| V1.7 | A thread root ID (`ParentMessageId` value) should NOT appear in `MessageIds` unless the root message's own content directly grounds part of the answer | Remove the root ID from `MessageIds` and replace with the specific reply IDs that contain the grounding content |
| V1.8 | `MessageIds` must point to the **original source message**, not a reply that merely quotes it via `<blockquote>`. If reply X quotes message Y, and the answer comes from the quoted content, `MessageIds` must include Y (the original), not X | Replace the quoting reply ID with the original message ID; include the quoting reply only if it adds new information beyond the quote |
| V1.9 | Every ID in `MessageIds` must contribute grounding content to `ExpectedAnswer`. No filler IDs | Remove any `MessageIds` entry whose content does not contribute to `ExpectedAnswer` |
| V1.10 | `RecencyCategory` must correctly reflect whether all `ParentMessageIds` are in the Recent50 set, none are, or a mix | Recalculate based on the Recent50 thread set derived from `message.createdDateTime` sort |
| V1.11 | Both `ParentMessageIds` and `MessageIds` fields end with a trailing `\|` and are quoted (e.g. `"1762804137000\|"`) | Append trailing `\|` and ensure field is quoted; never write a bare unquoted integer ID |

### §9.2 Pass 2 — Answer & Negative Accuracy Validation

For each **positive** row (`IsAnswerable=yes`), verify:

| Check | Rule | Fix if violated |
|-------|------|-----------------|
| V2.1 | `ExpectedAnswer` is factually supported by the content of the referenced `MessageIds` | Correct answer to match message content |
| V2.2 | `ExpectedAnswer` does not contain information absent from referenced messages | Remove unsupported claims |
| V2.3 | `Question` is answerable from the referenced messages (not just tangentially related) | Regenerate question or update MessageIds |
| V2.4 | `QuestionCategory` correctly matches the question type per §3 definitions | Reassign to correct category |
| V2.9 | Each `MessageId` in `MessageIds` contributes distinct, necessary content to `ExpectedAnswer` — verify by reading the actual message content from the JSON | Remove IDs whose message content is not reflected in the answer, or update `ExpectedAnswer` to include the content |
| V2.10 | Every factual claim in `ExpectedAnswer` must be traceable to the content of at least one referenced `MessageId`. If the answer contains claims not present in any referenced message's content, the grounding is incomplete | Add the correct `MessageId` whose content supports the ungrounded claim, or remove the unsupported claim from `ExpectedAnswer` |
| V2.11 | `ExpectedAnswer` must preserve tense and conditionality from the source. A source statement phrased as a future plan or milestone (e.g. "target: Private Preview Jan 2026") must not be rewritten as a current fact (e.g. "The team is in Private Preview") | Revert to the source framing — preserve "planned", "target", "expected", etc. |
| V2.12 | Named artifacts (bug titles, ADO work item titles, PR descriptions, feature names, error strings) must appear in `ExpectedAnswer` exactly as written in the source, character-for-character. Any compression or rephrasing of a named artifact is a C1 violation | Replace the paraphrased artifact name with the verbatim text from the source message |

For each **negative** row (`IsAnswerable=no`), verify:

| Check | Rule | Fix if violated |
|-------|------|-----------------|
| V2.5 | Question has **no topical relation** to any message in the input file | Replace with a completely unrelated question |
| V2.6 | Question does not contain any distinctive names, terms, tools, or concepts from the channel | Replace terms or regenerate question |
| V2.7 | `ExpectedAnswer`, `ParentMessageIds`, `MessageIds`, `ChunkCategory`, `QuestionCategory`, `Likelihood` are all empty | Clear any non-empty fields |
| V2.8 | Question is realistic and specific (not generic or absurd) | Rewrite to be more concrete and workplace-realistic |

### §9.3 Pass 3 — Distribution & Completeness Validation

> Covers **standard questions only** (20 positive `standard` + 20 negative `standard`).

| Check | Rule | Fix if violated |
|-------|------|-----------------|
| V3.1 | Standard positive count = 20, standard negative count = 20 | Add or remove questions |
| V3.2 | Exactly 20 rows with `IsAnswerable=yes, QuestionType=standard` and 20 with `IsAnswerable=no` | Rebalance |
| V3.3 | **Standard Positive:** `Likelihood` distribution: ~14 High, ~5 Medium, ~1 Rare | Reassign likelihood ratings |
| V3.4 | **Standard Positive:** No category has fewer than 1 question | Generate additional questions for underrepresented categories |
| V3.5 | **Standard Positive:** No category has more than 4 questions | Reassign or replace excess questions |
| V3.6 | **Standard Positive:** `single` chunk is 55-65%, `multi` chunk is 35-45% | Reassign or regenerate to meet targets |
| V3.11 | **Standard Positive:** Every `summary_compilation` and `status_and_progress` question has `ChunkCategory=multi` | Change `ChunkCategory` to `multi`; if the question cannot genuinely span multiple threads, regenerate it so that it does |
| V3.7 | No duplicate or near-duplicate questions across the standard 40 (>80% token overlap) | Replace duplicates |
| V3.8 | **Standard Positive:** Questions reference threads across the breadth of the channel | Generate replacement questions targeting underrepresented threads |
| V3.9 | **Negative:** Questions are topically diverse (not all from one domain) | Replace to diversify |
| V3.10 | **Standard Positive:** `RecencyCategory` distribution is ~4 Recent50, ~12 OlderThanRecent50, ~4 Mix | Reassign questions or regenerate to meet targets |

### §9.4 Pass 4 — Vector Search Positive Validation

> Covers the 20 vector search positive questions (`IsAnswerable=yes`, `QuestionType=vector_search`).

| Check | Rule | Fix if violated |
|-------|------|-----------------|
| V4.1 | Total vector search positive count = 20 | Add or remove to reach target |
| V4.2 | Every question avoids key terms from its referenced source messages (minimum 2 distinctive term substitutions) | Rewrite question to replace channel-specific terms with semantically equivalent alternatives |
| V4.3 | No vector search positive question is a near-duplicate of a standard positive question (>80% token overlap) | Rewrite the vector search question to approach the fact from a genuinely different angle |
| V4.4 | No vector search positive question is a near-duplicate of another vector search question | Replace duplicate |
| V4.5 | All grounding rules from §7.1 (ID validity, answer sourcing, parent-child correctness) hold | Apply the same ID fixes as Pass 1 |
| V4.6 | `Likelihood` distribution for vector search positives: ~14 High, ~5 Medium, ~1 Rare | Reassign likelihood ratings |
| V4.7 | `ChunkCategory` distribution: 55–65% single, 35–45% multi | Reassign or regenerate to meet targets |
| V4.8 | Every `summary_compilation` and `status_and_progress` vector search question has `ChunkCategory=multi` | Correct or regenerate |
| V4.9 | `RecencyCategory` distribution: ~4 Recent50, ~12 OlderThanRecent50, ~4 Mix | Reassign or regenerate to meet targets |
| V4.10 | All 11 question categories are represented (1–4 questions each) | Generate additional questions for underrepresented categories |
| V4.11 | Total CSV row count = 60 (40 positive + 20 negative) | Reconcile counts |

---

## §10 Agent Execution Procedure

The agent should follow these steps in order:

### Step 1: Ingest & Understand
1. Read the entire input JSON file
2. Count threads, unique authors, date range, and primary topics
3. Build a mental model of the channel's purpose and key discussion themes
4. Identify the most substantive threads (those with many replies, technical depth, or decision points)
5. **Build an exclusion inventory** — list all distinctive terms, names, project names, tool names, feature flags, skill names, and topics present in the channel (used to ensure negative questions have zero overlap)

### Step 2: Generate Standard Positive Questions (20, `QuestionType=standard`)
1. Iterate through threads systematically
2. For each substantive thread, generate candidate questions across applicable categories
3. For multi-chunk questions, identify threads that share related topics and formulate cross-thread questions
4. Assign `ChunkCategory`, `Likelihood`, `ParentMessageIds`, and `MessageIds` for each question
5. Write `ExpectedAnswer` by extracting and paraphrasing from the specific referenced messages
6. Target the distribution goals from §3.1, §4.1, and §5

### Step 3: Generate Vector Search Positive Questions (20, `QuestionType=vector_search`)
1. For each of the 20 standard positive questions, identify the underlying fact it targets
2. Reformulate the question using synonyms, conceptual rewrites, or broader/narrower phrasing — substituting at minimum 2 distinctive domain terms per §14.2
3. Verify the paraphrased question remains fully answerable from the same `MessageIds` (do not change grounding)
4. Ensure the paraphrase is not a near-duplicate of the standard question it is based on; approach the fact from a meaningfully different angle
5. Assign the same `ChunkCategory`, `Likelihood`, `ParentMessageIds`, `MessageIds`, `ExpectedAnswer`, and `RecencyCategory` as the corresponding standard question unless the paraphrase naturally maps to a different fact or scope
6. Target the distribution goals from §14.4

### Step 4: Generate Negative Questions (20, `QuestionType=standard`)
1. Using the exclusion inventory from Step 1, generate 20 realistic workplace questions on completely unrelated topics
2. Cross-check every negative question against the exclusion inventory — ensure zero term/topic overlap
3. Ensure topical diversity across the 20 negative questions
4. Set all metadata fields to empty per §8.3

### Step 5: Validate (4 passes)
1. Execute Pass 1 (§9.1) — fix all ID issues on standard positive questions
2. Execute Pass 2 (§9.2) — fix answer accuracy on standard positives, verify isolation of negatives
3. Execute Pass 3 (§9.3) — fix all distribution issues across the standard 40
4. Execute Pass 4 (§9.4) — validate all 20 vector search positive questions
5. After each pass, log the count of corrections made

### Step 6: Output
1. Write the final CSV per §6 formatting rules (standard positives rows 1–20, vector search positives rows 21–40, negatives rows 41–60)
2. Report summary statistics:
   - Standard positive: questions per category, single vs. multi chunk, likelihood distribution, total threads referenced
   - Vector search positive: questions per category, single vs. multi chunk, likelihood distribution, paraphrase substitution summary
   - Negative: count, topical diversity summary
   - Corrections made per validation pass

---

## §11 Example Questions by Category

These examples are based on the `TS Orches MCP A2A.json` channel to illustrate expected output quality:

### Positive: `information_gathering` (single)
> **Q:** What was the 'additionalProperties' fix that Bhawna proposed for the meeting_scheduler schema issue?  
> **A:** Bhawna proposed adding 'additionalProperties:false/true' to the skill schema in the scheduling-skill-adapter.ts file. The issue occurred because when 'enableToolsStrictMode' is true, the schema must be strict and requires 'additionalProperties' to be set.  
> **MessageIds:** 1762805832000|1762842734000

### Positive: `decision_and_action_history` (single)
> **Q:** What was decided about the enableParticipantArraySchema flag after the LLM errors were discovered on Nov 10?  
> **A:** Vijay asked Victoria to validate the flow and enable the flag. Bhawna submitted ECS PR 1387005 to disable Task Update v2 from R0 until the fix reached PROD. The fix was merged and would re-enable after reaching production.  
> **MessageIds:** 1762814691000|1762808584000|1762809944000

### Positive: `risk_and_issue_identification` (single)
> **Q:** What risk did Volodymyr identify with the planner failure final status showing as "Success"?  
> **A:** Volodymyr identified that when the planner fails all iterations, it only has a final response with status "Success", meaning these failures can go unnoticed since monitoring checks final status. He asked Mohammad and Vijay whether there was an alarm for planner failures specifically.  
> **MessageIds:** 1762812312000

### Positive: `issue_compilation` (multi)
> **Q:** List all the LLM-related errors and their affected skills discussed in the LLM Errors thread on Nov 10.  
> **A:** Two skills had schema validation errors: (1) 'manage_tasks' — 6 affected users with a combined 144 planner failures, and (2) 'meeting_scheduler' — 2 affected users (Junaid and Bhawna) with 72 combined planner failures. Additionally, token limit issues were identified coming from the planner, and "too big request" errors were observed in DF traffic.  
> **MessageIds:** 1762817545000|1762817662000|1762805068000|1762805275000

### Negative Examples
> - "What was the final headcount approved for the Austin office expansion?"
> - "When is the next all-hands meeting for the Azure DevOps marketing team?"
> - "Who approved the vendor contract for the new cafeteria management system?"
> - "What are the updated guidelines for submitting expense reports in Concur?"

---

## §12 Error Handling

| Scenario | Agent Behavior |
|----------|---------------|
| Thread has 0 replies | Skip for question generation unless root message alone is informative |
| Message content is only an image/emoji/file attachment | Skip — cannot generate grounded questions from non-text content |
| Thread is too short for meaningful question | Use as supporting evidence in multi-chunk questions only |
| Cannot reach 20 positive questions | Generate maximum possible, document gap with reasoning |
| Cannot reach 20 negative questions | Generate maximum possible, document gap with reasoning |
| Cannot meet distribution targets exactly | Get as close as possible, document any deviations with counts |
| Ambiguous category assignment | Assign to the most specific matching category |

---

## §13 Success Criteria

- [ ] Output CSV has exactly 60 rows (excluding header): 40 positive + 20 negative
- [ ] All positive rows have `IsAnswerable=yes` with all fields populated
- [ ] All negative rows have `IsAnswerable=no` with `QuestionCategory`, `ChunkCategory`, `ExpectedAnswer`, `ParentMessageIds`, `MessageIds`, `Likelihood` empty
- [ ] **Standard Positive:** All 11 question categories are represented (1–4 each)
- [ ] **Standard Positive:** Likelihood distribution is ~14 High / ~5 Medium / ~1 Rare
- [ ] **Standard Positive:** Single/multi chunk split is within 55-65% / 35-45%
- [ ] **Standard Positive:** Every `summary_compilation` and `status_and_progress` question has `ChunkCategory=multi`
- [ ] **Standard Positive:** Every `ParentMessageId` and `MessageId` exists in the input JSON
- [ ] **Standard Positive:** Every `ExpectedAnswer` is factually supported by its referenced messages
- [ ] **Negative:** Every question has zero topical/keyword overlap with channel content
- [ ] **Negative:** Questions are topically diverse and realistic
- [ ] No duplicate or near-duplicate questions across all 60
- [ ] 4 validation passes completed with correction counts reported
- [ ] Positive questions span the breadth of channel topics and timeframe
- [ ] **Standard Positive:** `RecencyCategory` is correctly computed for every positive question based on the Recent50 thread set
- [ ] CSV is valid (parseable by standard CSV readers, proper quoting/escaping)
- [ ] **Vector Search Positive:** 20 rows with `IsAnswerable=yes` and `QuestionType=vector_search`
- [ ] **Vector Search Positive:** Every question demonstrably avoids the key terms of its referenced source messages (minimum 2 distinctive term substitutions)
- [ ] **Vector Search Positive:** All 11 question categories are represented (1–4 each); likelihood distribution is ~14/~5/~1; single/multi split is 55–65% / 35–45%
- [ ] **Vector Search Positive:** Every `summary_compilation` and `status_and_progress` question has `ChunkCategory=multi`
- [ ] **Vector Search Positive:** All grounding rules from §7.1 apply (real IDs, sourced answers, correct RecencyCategory)
- [ ] **Vector Search Positive:** No question duplicates or near-duplicates an existing standard positive question

---

## §14 Vector Search Positive Question Generation (`QuestionType=vector_search`)

### §14.1 Purpose

Vector search positive questions stress-test the **semantic retrieval layer** of the RAG system. Each question expresses an information need that is present in the channel, but using **different vocabulary** from the source message. The goal is to verify that the vector search index retrieves the correct chunk via embedding similarity even when there is no keyword overlap between the query and the source text.

These 20 questions are tagged `QuestionType=vector_search` and occupy rows 21–40 in the output CSV.

| Block | `QuestionType` | `IsAnswerable` | Count | CSV rows |
|-------|----------------|----------------|-------|----------|
| Standard positive | `standard` | `yes` | 20 | 1–20 |
| Vector search positive | `vector_search` | `yes` | 20 | 21–40 |
| Standard negative | `standard` | `no` | 20 | 41–60 |

---

### §14.2 Paraphrase Requirements

1. **Avoid key terms from the source message.** The question must not reuse the same distinctive nouns, verb phrases, technical identifiers, or domain-specific terms that appear verbatim in the referenced messages.

2. **Substitute with semantically equivalent alternatives.** Replace **at minimum two distinctive domain-specific terms** with synonyms, broader concepts, or conceptual rewrites. Examples:

   | Source phrase | Acceptable paraphrase |
   |---------------|-----------------------|
   | `schema validation error` | "malformed tool definition issue" / "invalid structure in the function spec" |
   | `disable the flag` | "turn off the feature toggle" / "deactivate the configuration switch" |
   | `merged the PR` | "when the code change was integrated" / "after the patch was accepted" |
   | `planner failure` | "orchestration loop breakdown" / "task execution pipeline fault" |
   | `R0 users` | "early-ring testers" / "initial rollout audience" |
   | `enableToolsStrictMode` | "the strict tool schema enforcement setting" |
   | `additionalProperties` | "the extra-fields permission in the JSON schema" |

3. **Remain fully answerable.** Despite the different phrasing, the question must still be answerable from the same referenced messages. All grounding rules from §7.1 apply without modification.

4. **Not be a trivial paraphrase.** Simply reordering words, changing "what" to "which", or adding/removing "please" does not qualify. The substitution must make the question materially harder to match via lexical overlap.

5. **Not duplicate a standard positive question.** A vector search question may target the same underlying fact as a standard question, but it must approach it from a sufficiently different angle — not just a word-swapped copy of an existing standard question.

---

### §14.3 All Other Rules

- All rules from §7.1 (Grounding Requirements) apply in full — real IDs, sourced answers, correct parent-child ID relationships.
- All rules from §7.2 (Question Quality Requirements) apply — specific, naturally phrased, varied styles, no duplicates.
- HTML content handling per §7.3 applies.
- The `ExpectedAnswer` is written in **natural source language** (not paraphrased) — the paraphrase constraint applies to the *question* only, not the answer.

---

### §14.4 Distribution Targets (20 vector search positive questions)

| Attribute | Target |
|-----------|--------|
| Question categories | All 11 represented; no fewer than 1, no more than 4 per category |
| Likelihood | ~14 High, ~5 Medium, ~1 Rare |
| ChunkCategory | 55–65% single (11–13 questions), 35–45% multi (7–9 questions) |
| RecencyCategory | ~4 Recent50, ~12 OlderThanRecent50, ~4 Mix |
| `summary_compilation` / `status_and_progress` | Always `ChunkCategory=multi` |

---

### §14.5 Field Values for Vector Search Positive Questions

All fields are identical to a standard positive question (§6), with:

| Field | Value |
|-------|-------|
| `IsAnswerable` | `yes` |
| `QuestionCategory` | One of the 11 category IDs from §3 |
| `ChunkCategory` | `single` or `multi` |
| `Question` | Paraphrased question text (no key source terms) |
| `ExpectedAnswer` | Concise answer sourced from referenced messages |
| `ParentMessageIds` | Pipe-delimited thread IDs |
| `MessageIds` | Pipe-delimited message IDs |
| `Likelihood` | `High`, `Medium`, or `Rare` |
| `RecencyCategory` | `Recent50`, `OlderThanRecent50`, or `Mix` |
| `QuestionType` | `vector_search` |

---

### §14.6 Example Vector Search Positive Questions

> **Standard question:** "What was the `additionalProperties` fix that Bhawna proposed for the meeting_scheduler schema issue?"
>
> **Vector search equivalent:** "What structural correction to the scheduling component's function definition did the engineer suggest to resolve the strict-mode incompatibility?"
>
> *(Same MessageIds, same ExpectedAnswer — only the question vocabulary differs)*

---

> **Standard question:** "What decision was made about disabling the flag in prod after the LLM errors were discovered?"
>
> **Vector search equivalent:** "What action was taken to turn off the configuration switch in the production environment after the model response failures were identified?"

---

### §14.7 Anti-Patterns (Bad Vector Search Positive Questions)

| Bad Example | Why it's bad |
|-------------|--------------|
| "Which `enableToolsStrictMode` setting caused the issue?" | Reuses an exact channel term — not a valid paraphrase |
| "What was the error in the meeting_scheduler?" | Only rephrases the question structure, not the domain terms |
| Question that is NOT answerable from any channel message | Violates grounding — vector search questions must still be answerable |
| Near-identical copy of an existing standard question with 1 word changed | Does not represent a genuine vocabulary shift |
