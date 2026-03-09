# Question-Answer Content Relevance Validation Spec

## §1 Overview

Validates that each **positive row** (`IsAnswerable=yes`) in the questions CSV has its `Question` and `ExpectedAnswer` grounded in the actual message content referenced by `ParentMessageIds` and `MessageIds`. Also checks that no referenced ID is irrelevant to the answer.

**Negative rows (`IsAnswerable=no`) are skipped entirely — no validation needed.**

| Item | Detail |
|------|--------|
| Executor | LLM agent using `grep_search`, `read_file`, `replace_string_in_file` only — no Python/scripts |
| Input CSV | `LoadedChannelDataDump/<channel_name>_questions.csv` |
| Input JSON | `LoadedChannelDataDump/<channel_name>.json` |
| Output | Same CSV with `ValidationResult` column appended |

---

## §2 JSON Structure (Quick Reference)

```
Thread: { "id": "<ParentMessageId>", "message": { "content": "...", ... }, "replies": { "messages": [ { "id": "<MessageId>", "content": "...", "replyToId": "<ParentMessageId>", ... } ] } }
```

---

## §3 Validation Checks (Positive Rows Only)

For each row where `IsAnswerable=yes`, run these checks. First failure wins. C4 applies only to `QuestionType=vector_search` rows.

| Check | What | How | Failure Value |
|-------|------|-----|---------------|
| **C1** | **Answer grounded in referenced messages** | Grep JSON for each `ParentMessageId` and `MessageId`. Read the `content` fields of all matched messages. Verify the key claims in `ExpectedAnswer` (names, IDs, actions, decisions, numbers, tool/file names) appear **verbatim or as direct literal equivalents** in those message contents. Do **NOT** accept approximate, inferred, paraphrased, or topic-proximity matches — each claim must be quotable word-for-word from the source text. | `FAIL:C1:<brief_reason>` |
| **C2** | **No irrelevant ID referenced** | For each `ParentMessageId` and `MessageId`, check that its message content is actually relevant to the `Question` and `ExpectedAnswer`. If any referenced ID's content has no bearing on the Q&A, flag it. | `FAIL:C2:irrelevant_id:<ID>` |
| **C3** | **`summary_compilation` and `status_and_progress` use `multi` chunk only** | If `QuestionCategory` is `summary_compilation` or `status_and_progress`, verify `ChunkCategory` is `multi`. | `FAIL:C3:single_chunk_not_allowed_for:<QuestionCategory>` |
| **C4** *(vector_search rows only)* | **Question avoids verbatim source terms (paraphrase quality check)** | Only runs when `QuestionType=vector_search`. Read the `content` of all referenced messages (same content already fetched for C1). Extract the distinctive domain-specific terms: technical identifiers, product names, feature flag names, skill names, file names, error message strings, and API names. Verify that the `Question` field does **NOT** reuse 2 or more of these terms verbatim. Common stopwords, generic verbs (e.g. "what", "how", "did", "was", "the") and author first names are exempt — only count distinctive technical/domain nouns. If 2 or more distinctive source terms appear word-for-word in the question → fail. This is the inverse of C1: where C1 requires verbatim presence in the answer, C4 requires verbatim absence in the question. | `FAIL:C4:reuses_source_terms:<term1>,<term2>` |

Rows passing all checks get `PASS`.

---

## §4 Agent Workflow

### §4.1 Per-Row Steps

For each positive row:

1. Parse `ParentMessageIds` (pipe-delimited) and `MessageIds` (pipe-delimited) from the row. Fields end with a trailing `|` (e.g. `"1762804137000|"`) — after splitting on `|`, **discard empty tokens** so only real IDs remain.
2. For each ID, `grep_search` the JSON file for `"id": "<ID>"`.
3. `read_file` the matched region (±15 lines) to capture the `content` field.
4. Collect all message content (strip HTML tags mentally).
5. **C1**: For each key claim in `ExpectedAnswer`, locate the **exact quote** from a cited message that supports it. If no verbatim or direct literal match can be found in any cited message content → `FAIL:C1`. Do NOT accept semantic similarity, paraphrase, or topic proximity as a match.
6. **C2**: For each referenced ID, check its content is relevant to the Q&A. If any ID's content is unrelated → `FAIL:C2`.
7. **C3**: If `QuestionCategory` is `summary_compilation` or `status_and_progress`, verify `ChunkCategory` is `multi`. If it is `single` → `FAIL:C3`.
8. **C4** *(only if `QuestionType=vector_search`)*: From the already-collected message content, extract distinctive domain-specific terms (technical identifiers, feature flag names, skill/API names, error strings, file names — excluding stopwords and generic verbs). Count how many of these appear verbatim in the `Question` field. If **2 or more** distinctive source terms appear word-for-word in the question → `FAIL:C4:reuses_source_terms:<term1>,<term2>`. Skip this check entirely for `QuestionType=standard` rows.
9. If all applicable checks pass → `PASS`.

### §4.2 Efficiency Tips

- **Group rows by `ParentMessageId`** — read each thread once, validate all rows referencing it.
- **Reuse content** — don't re-grep the same ID across rows.

---

## §5 Output

### §5.1 CSV Update

Append `ValidationResult` as the last column. Update the CSV in-place (header + each positive data row). Negative rows get `SKIP` as their validation result.

### §5.2 Summary

Print after completion:

```
=== VALIDATION SUMMARY ===
Positive rows:         <N>  (standard: <N>, vector_search: <N>)
Passed:                <N>
Failed:                <N>
  C1 (answer not grounded):                        <N>
  C2 (irrelevant ID):                              <N>
  C3 (single chunk for multi-only cat):            <N>
  C4 (question reuses source terms) [VS only]:     <N>
Skipped (negative rows):    <N>
```

---

## §6 Invocation

```
Follow the spec in LoadedChannelDataDump/question_validation_spec.md
to validate: LoadedChannelDataDump/<channel_name>_questions.csv
against: LoadedChannelDataDump/<channel_name>.json
```

Constraints: no Python, no scripts, grep/read/edit tools only, modify CSV in-place, process all positive rows.
