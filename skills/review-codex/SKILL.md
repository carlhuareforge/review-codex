---
name: review-codex
disable-model-invocation: true
description: >
  Multi-turn adversarial design review on .md files. Claude is the
  designer with full context. Codex (gpt-5.4) is shelled out as
  reviewer with full workspace access. Comments logged to markdown-threads
  v2.0 .comments.json sidecar files, viewable in VS Code.
---

# Adversarial Design Review

## Trigger

User says "review", "design review", "adversarial review", or similar
followed by a path to a `.md` file.

## Overview

Claude is the **designer** — holds full codebase context, conversation
history, and design reasoning. Codex is the **reviewer** — shelled out
as a subprocess with full workspace access. It explores real code, verifies
claims, and delivers grounded critique using `gpt-5.4-xhigh`.

**Authentication:** Codex CLI uses ChatGPT subscription auth (not API keys).
The user must run `codex login` once to authenticate via browser. Verify
with `codex login status` before starting. If not logged in, prompt the
user to run `codex login`.

**Session continuity:** Across multi-round reviews, Codex maintains context
via session resume. Round 1 creates a new session; rounds 2+ resume that
session with `codex exec resume <SESSION_ID>` so the reviewer remembers
its prior exploration, comments, and the designer's rebuttals.

Context isolation is critical: Codex's exploration noise (file reads,
grep output, reasoning traces) never enters Claude's context window.
Only the final structured JSON output (read from `.review-output.json`)
enters Claude's context.

## .comments.json Schema (v2.0)

The sidecar file follows the markdown-threads VS Code extension v2.0
schema exactly. This is non-negotiable — the extension validates it.

```json
{
  "doc": "design.md",
  "version": "2.0",
  "comments": [
    {
      "id": "uuid-v4",
      "anchor": {
        "sectionSlug": "authentication-flow",
        "lineHint": 14,
        "selectedText": "OAuth2 is used for all external API calls"
      },
      "status": "open",
      "thread": [
        {
          "id": "uuid-v4",
          "author": "codex-reviewer",
          "body": "Comment text here.",
          "created": "2026-03-19T10:30:00.000Z"
        }
      ]
    }
  ]
}
```

### Field rules

- `doc`: filename only (e.g. `"design.md"`, not a path)
- `version`: must be exactly `"2.0"`
- `comments[]`: array of CommentThread objects
- `CommentThread.id`: UUID v4
- `CommentThread.anchor.sectionSlug`: slugified nearest heading above the issue
- `CommentThread.anchor.lineHint`: 0-indexed line number
- `CommentThread.anchor.selectedText`: the exact text being commented on (optional but strongly recommended)
- `CommentThread.status`: `"open"` or `"resolved"`
- `CommentThread.thread[]`: array of CommentEntry objects (at least one)
- `CommentEntry.id`: UUID v4
- `CommentEntry.author`: string identifier (use `"codex-reviewer"` for reviewer, `"claude-designer"` for designer)
- `CommentEntry.body`: comment text (markdown supported)
- `CommentEntry.created`: ISO-8601 timestamp

### Slugify algorithm

```
1. Lowercase the heading text
2. Trim whitespace
3. Remove non-word characters (keep letters, numbers, spaces, hyphens)
4. Replace spaces with hyphens
5. Collapse consecutive hyphens
6. Trim leading/trailing hyphens
```

Examples: `"Authentication Flow"` → `"authentication-flow"`, `"API v2.0 Endpoints"` → `"api-v20-endpoints"`

## Behavior

### Step 0: Initialization

1. **Parse invocation.** Extract the `.md` file path from the user's message.

2. **Validate Codex CLI.**
   - Run `which codex`. If not found, check `~/.npm-global/bin/codex` and
     common paths. If still not found, abort:
     `"Codex CLI not found. Install with: npm install -g @openai/codex"`
   - Run `codex login status`. If not logged in, abort:
     `"Codex is not authenticated. Run: codex login"`

3. **Read the target .md file** into working memory.

4. **Read or create the sidecar.** If `<filename>.comments.json` exists in the
   same directory, read it. If it exists but is malformed, back it up to
   `.comments.json.bak` and create fresh. If it doesn't exist, create:
   ```json
   {
     "doc": "<filename>.md",
     "version": "2.0",
     "comments": []
   }
   ```

5. **Read config.** Load `.review-config.json` from the document's directory
   if present. Otherwise use defaults:
   ```json
   {
     "maxRounds": 3,
     "convergenceThreshold": 0,
     "designerAuthor": "claude-designer",
     "reviewerAuthor": "codex-reviewer",
     "reviewerModel": "gpt-5.4-xhigh",
     "relevantPaths": []
   }
   ```

6. **Set round counter and session tracking.**
   - `current_round = 1`
   - `codex_session_id = null` (will be captured after round 1)
   - `consecutive_failures = 0`

### Step 1: Reviewer Turn (each round)

7. **Build reviewer prompt.** Write `.review-prompt.md` in the document's
   directory containing:
   - Role instructions (adversarial reviewer, explore codebase first)
   - The full design doc content
   - Serialized thread history from previous rounds (human-readable)
   - Output format specification (JSON array to `.review-output.json`)
   - Relevant file paths from config

   Use the template in [Reviewer Prompt Template](#reviewer-prompt-template).

8. **Invoke Codex.** stdout/stderr go to /dev/null — context isolation.

   **Round 1 — new session:**
   ```bash
   cd "<document-directory>" && \
   codex exec \
     --dangerously-bypass-approvals-and-sandbox \
     -c 'model="gpt-5.4"' \
     -c 'model_reasoning_effort="xhigh"' \
     -c 'service_tier="fast"' \
     -C "<document-directory>" \
     --json \
     "Read the file .review-prompt.md for your instructions. \
      You have full access to this workspace. Explore the codebase \
      as needed to ground your review. Write your final output ONLY \
      to the file .review-output.json. Do not print it to stdout." \
     2>/dev/null | tee .codex-session.jsonl > /dev/null
   ```

   After round 1 completes, extract the session ID from the JSONL output:
   ```bash
   CODEX_SESSION_ID=$(grep -m1 '"thread_id"' .codex-session.jsonl | \
     sed 's/.*"thread_id"\s*:\s*"\([^"]*\)".*/\1/')
   ```

   **Round 2+ — resume existing session:**
   ```bash
   cd "<document-directory>" && \
   codex exec resume "$CODEX_SESSION_ID" \
     --dangerously-bypass-approvals-and-sandbox \
     -c 'model="gpt-5.4"' \
     -c 'model_reasoning_effort="xhigh"' \
     -c 'service_tier="fast"' \
     --json \
     "Read the updated .review-prompt.md — it contains the designer's \
      responses to your previous comments. Review the rebuttals, push \
      back where unconvinced, concede where sound, and raise any new \
      issues. Write output to .review-output.json." \
     2>/dev/null | tee .codex-session.jsonl > /dev/null
   ```

   If session resume fails (e.g. session expired), fall back to a new session
   and include full thread history in the prompt so no context is lost.

   **Important flags:**
   - `--dangerously-bypass-approvals-and-sandbox`: no approval prompts, full
     filesystem access for deep code exploration
   - `-c 'model="gpt-5.4"'`: enforce gpt-5.4 model
   - `-c 'model_reasoning_effort="xhigh"'`: maximum reasoning depth
   - `-c 'service_tier="fast"'`: fast inference throughput (subscription only)
   - `-C "<dir>"`: working directory for the reviewer
   - `--json`: emit JSONL events (for session ID capture)
   - `2>/dev/null | tee .codex-session.jsonl > /dev/null`: stderr discarded,
     JSONL captured to file for session ID extraction, nothing in Claude's context

9. **Read reviewer output.** Read `.review-output.json`. This is the ONLY
   artifact from the reviewer that enters Claude's context.

10. **Handle missing output.** If `.review-output.json` doesn't exist:
    - Increment `consecutive_failures`
    - If `consecutive_failures >= 2`, abort the review
    - Otherwise log a warning and skip to next round

11. **Parse and validate.** Expect a JSON array of comment objects:
    ```json
    [
      {
        "headingSlug": "section-slug",
        "body": "Issue description with evidence.",
        "severity": "high",
        "selectedText": "the exact text being commented on",
        "replyToThreadId": null,
        "filesReferenced": ["src/file.ts:L42"]
      }
    ]
    ```
    If malformed (not valid JSON or not an array), wrap the raw text as a
    single comment anchored to `_document_root` with lineHint 0.
    Reset `consecutive_failures = 0` on successful parse.

12. **Write reviewer comments to .comments.json.** For each comment in the
    array:
    - If `replyToThreadId` is set and matches an existing thread, append a
      new CommentEntry to that thread
    - Otherwise create a new CommentThread with:
      - `id`: new UUID v4
      - `anchor.sectionSlug`: from `headingSlug`
      - `anchor.lineHint`: find the 0-indexed line number of that heading in the .md
      - `anchor.selectedText`: from `selectedText` if provided
      - `status`: `"open"`
      - `thread[0]`: CommentEntry with author = config.reviewerAuthor,
        body = the comment body (prepend severity as bold: `**[HIGH]** ...`),
        created = ISO-8601 now

13. **Clean up temp files (keep session file):**
    ```bash
    rm -f .review-prompt.md .review-output.json
    ```
    Keep `.codex-session.jsonl` — needed for session resume in subsequent rounds.

### Step 2: Designer Turn (each round)

14. **Read the updated .comments.json.** Claude now sees all reviewer
    comments from this round.

15. **Respond using full native context.** For each new reviewer comment
    (from the current round), Claude decides:
    - **Accepted:** valid issue, note what needs to change
    - **Rebut:** the design is correct, here's why
    - **Clarify:** additional context resolves the ambiguity
    - **Resolve:** mark thread status as `"resolved"`

16. **Write designer replies to .comments.json.** For each response:
    - Append a new CommentEntry to the relevant thread
    - author = config.designerAuthor
    - body = the response text
    - created = ISO-8601 now
    - If resolving, set thread status to `"resolved"`

17. **Increment round.** `current_round += 1`.

### Step 3: Convergence Check

18. **After each complete round, check stopping conditions (in order):**
    - `current_round > maxRounds` → stop
    - All threads have status `"resolved"` → stop (early convergence)
    - Reviewer returned empty array `[]` → stop
    - Count of open threads ≤ `convergenceThreshold` → stop

19. **If not converged:** go to step 7. The reviewer resumes its session
    and sees the designer's rebuttals. It can push harder, concede, or
    raise new emergent issues.

### Step 4: Finalization

20. **Clean up all temp files:**
    ```bash
    rm -f .review-prompt.md .review-output.json .codex-session.jsonl
    ```

21. **Print terminal summary:**

    ```
    ═══ Adversarial Review Complete ═══
    Document:  <path>
    Rounds:    N / maxRounds
    Designer:  claude (native)
    Reviewer:  codex
      Model:           gpt-5.4
      Reasoning:       xhigh
      Service tier:    fast
      Session ID:      <session-uuid>

    Threads:   T total
      Accepted:    A
      Resolved:    R
      Open:        O (breakdown by severity)

    Open issues:
      § <Section Heading> — SEVERITY
        <brief description>

    Full history: <path>.comments.json
    View in VS Code: install markdown-threads extension
    ```

## Reviewer Prompt Template

Written to `.review-prompt.md` before each Codex invocation. The design
doc is NOT embedded — Codex reads it directly from the workspace. Only
the role instructions, output format, and thread history are included.

````markdown
# Design Review Request

You are an adversarial design reviewer. Your job is to find flaws,
inconsistencies, missing edge cases, unstated assumptions, and logical
gaps in a design document.

## Target Document

Read `{DESIGN_DOC_FILENAME}` in this directory. That is the document
under review. Do NOT rely on any cached version — read it fresh.

## Before You Comment

You have FULL ACCESS to this workspace. USE IT.

- Read the design doc thoroughly first
- Explore source files referenced in the design
- Verify that claimed implementations exist and match
- Check constants, types, schemas against what the design assumes
- Look at tests — are the design's edge cases covered?
- If the design references external contracts or APIs, check them

Ground every comment in evidence. "Section 3 says the cliff is
12 months but src/vesting.sol:L42 sets CLIFF_DURATION to 180 days"
is useful. "The cliff might be wrong" is not.

## Suggested Paths (explore beyond these as needed)

{RELEVANT_FILE_PATHS}

## Rules

- Reference the exact section heading where the issue exists
- State severity: high | medium | low | question
- If previous rounds exist below, read the designer's responses.
  Push back if unconvinced, concede if sound, or raise new issues
  that emerged from the response.
- Do NOT repeat resolved issues
- Do NOT give general praise. Every comment identifies a problem
  or asks a question.
- When referencing code you explored, include the file path and
  line number in a "filesReferenced" array
- Include "selectedText" with the exact text from the design doc
  that your comment refers to

## Output

Write your output to the file `.review-output.json` in the current
working directory. Do NOT print it to stdout.

The file must contain ONLY a JSON array:

```json
[
  {
    "headingSlug": "token-unlock-schedule",
    "body": "Description of the issue with evidence from codebase.",
    "severity": "high",
    "selectedText": "the exact text you are commenting on",
    "replyToThreadId": null,
    "filesReferenced": ["src/vesting.sol:L42"]
  }
]
```

`replyToThreadId`: set to a thread ID to reply within an existing
thread, or null for a new top-level issue.

If you have no new issues or replies, write an empty array: `[]`

---

## Previous Review Threads

{SERIALIZED_THREAD_HISTORY_OR_"No previous rounds."}
````

## Critical Rules

- NEVER pipe Codex stdout into Claude's context. Always discard or redirect.
- ONLY read `.review-output.json` for reviewer results.
- NEVER modify existing human comments in `.comments.json`.
- NEVER modify the design doc during review.
- Delete temp files after each round (except session file until finalization).
- All UUIDs must be v4 format.
- `lineHint` is 0-indexed.
- `version` must be exactly `"2.0"`.
- Preserve any existing comments in the sidecar — only append.
- Always enforce model config via `-c` overrides (model, model_reasoning_effort, service_tier).
- Use `codex exec resume` for rounds 2+ to maintain reviewer context.

## Configuration

`.review-config.json` in the document's directory (optional):

```json
{
  "maxRounds": 3,
  "convergenceThreshold": 0,
  "designerAuthor": "claude-designer",
  "reviewerAuthor": "codex-reviewer",
  "reviewerModel": "gpt-5.4-xhigh",
  "relevantPaths": ["src/", "contracts/", "test/"]
}
```

| Field | Default | Description |
|---|---|---|
| `maxRounds` | `3` | Maximum review rounds before forced stop |
| `convergenceThreshold` | `0` | Stop when open threads ≤ this number |
| `designerAuthor` | `"claude-designer"` | Author string for designer comments |
| `reviewerAuthor` | `"codex-reviewer"` | Author string for reviewer comments |
| `reviewerModel` | `"gpt-5.4-xhigh"` | Codex model to use |
| `relevantPaths` | `[]` | Paths to suggest to reviewer |

## Edge Cases

- **Codex CLI not found.** Abort: `"Install with: npm install -g @openai/codex"`
- **Not logged in.** Abort: `"Run: codex login"`
- **Codex times out or crashes.** `.review-output.json` won't exist. Log a
  warning, skip to next round. Abort after 2 consecutive failures.
- **Session resume fails.** Fall back to new session with full thread history
  in the prompt — no context is lost, just a fresh session.
- **Reviewer returns prose instead of JSON.** Wrap as single comment on
  `_document_root`, lineHint 0.
- **Design doc has no headings.** Anchor all comments to `_document_root`.
- **Empty reviewer response `[]`.** Valid convergence signal. Stop.
- **Existing human comments.** Preserve unconditionally. Only append.
- **Malformed .comments.json.** Back up to `.bak`, create fresh.

## Human-in-the-Loop

After review completes, open the `.md` in VS Code with the markdown-threads
extension installed. Every agent comment appears as a threaded discussion.
You can reply, resolve, reopen, or add new threads.

Re-running the skill after modifying the design preserves existing threads.
The reviewer sees full history including human comments.
