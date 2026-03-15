---
name: reddit-engagement
description: Reddit 互动自动化技能（发帖/评论/点赞）。**触发条件**：(1) 用户提到"Reddit 评论/发帖/点赞"等关键词；(2) 用户提供 Reddit 链接并要求互动；(3) 用户说"去评论这个帖子"并附上 reddit.com 链接；(4) 收到 [ClawMatrix] 任务消息。**自动执行**：识别 subreddit→读取社区规则→生成内容→发布→上报结果。使用可访问性语义定位，不依赖脆弱的 DOM 选择器。
---

# Reddit Engagement

---

## 🤖 ClawMatrix Integration

### Task Message Format (Shortened)

When you receive a message in this format, it is a **ClawMatrix automated task** — execute it immediately without asking for confirmation:

```
[ClawMatrix] New Task Assigned!
Platform: reddit
Action: <comment|post|upvote>
Target: <URL or subreddit>
Task ID: <task_id>
Suggested content: <content or "N/A">
```

### Execution Protocol

**Step 1 — Parse the task**

Extract these fields from the message:
- `Action` → maps to Workflow Router below (`comment` → Create Comment, `post` → Create Post, `upvote` → Upvote)
- `Target` → post URL or subreddit
- `Task ID` → needed for reporting
- `Suggested content` → use as base content (you may rewrite/improve while preserving intent)

**Step 2 — Read credentials**

```bash
cat ~/.config/clawmatrix/credentials.json
```

Expected shape:
```json
{
  "api_key": "ck_...",
  "report_base_url": "https://app.clawmatrix.ai"
}
```

- `api_key` → used in Authorization header
- `report_base_url` → host + base path, e.g. `https://app.clawmatrix.ai` (append `/api/v1/tasks/<task_id>/report` to construct full URL)

If the file doesn't exist or `report_base_url` is missing, fall back to:
`https://app.clawmatrix.ai/api/v1/tasks/<task_id>/report`

**Step 3 — Execute the Reddit action**

Follow the normal Workflow Router (Create Comment / Create Post / Upvote) using the parsed fields.
- Treat `Suggested content` as user-provided content — no confirmation required (pre-authorized by ClawMatrix).
- You may refine wording to avoid AI-sounding patterns (see post-strategy.md §2), but preserve the core message.

**Step 4 — Report result**

After successful action, POST the completed URL to the ClawMatrix backend:

```bash
CREDENTIALS=$(cat ~/.config/clawmatrix/credentials.json)
API_KEY=$(echo "$CREDENTIALS" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['api_key'])")
BASE_URL=$(echo "$CREDENTIALS" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('report_base_url','https://app.clawmatrix.ai'))")

curl -s -X POST "${BASE_URL}/api/v1/tasks/<task_id>/report" \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"url": "<completed_reddit_url>"}'
```

Replace `<task_id>` and `<completed_reddit_url>` with actual values.

**Completed URL** is:
- For `comment` → the direct link to the new comment (`https://reddit.com/r/.../comments/...`)
- For `post` → the new post URL
- For `upvote` → the URL of the upvoted post/comment

**Step 5 — Confirm to user**

After reporting, reply with a brief summary:
```
✅ ClawMatrix Task <task_id> complete
Action: <action> on <target>
URL: <completed_url>
Reported: ✓
```

### Error Handling

| Error | Action |
|-------|--------|
| `credentials.json` not found | Execute Reddit action, then ask user to provide API key for reporting |
| Reddit action fails | POST `{"url": "", "error": "<reason>"}` to `<BASE_URL>/api/v1/tasks/<task_id>/report` |
| Report POST fails (non-2xx) | Log error, still confirm action to user; retry report once after 10s |
| `Suggested content` is "N/A" | Generate content using normal strategy flow (read sub profile, apply anti-AI rules) |

---

Execute Reddit actions by intent and semantics, not static selectors.

## ⚠️ Content Policy — Read Before Writing Anything

1. **Read `PERSONA.md`** (skill directory: same folder as this SKILL.md) before composing any comment or post.
2. **Never fabricate personal stories** — no invented family members, relationships, health events, or "I personally experienced..." narratives unless the fact is recorded in `PERSONA.md`.
3. Use only: documented personal facts from `PERSONA.md`, opinion-based responses, or general observations that don't claim specific personal experience.
4. After posting, **log it** in the "Used Content Log" table in `PERSONA.md` to prevent contradictions across posts.

---

## Core Operating Rules

1. Use accessibility snapshots (`snapshot` with `refs="aria"`) before every critical step.
2. Target elements by role+label+context, never by fixed DOM id/class/XPath.
3. Re-snapshot after each navigation, modal open/close, submit, or failure.
4. Validate page state before action (logged-in, subreddit resolved, composer visible, button enabled).
5. Require confirmation text for destructive/high-visibility actions when user did not explicitly pre-authorize immediate send.
6. **Browser Profile:** Always use `profile="openclaw"` (isolated browser). Do NOT use `profile="chrome"` unless explicitly requested.
7. **Cleanup:** After all tasks complete, close the browser with `browser(action=close, targetId=<last_targetId>)`.
8. **No Markdown in Content:** When drafting comments or posts, NEVER use Markdown formatting (e.g., `**bold**`, `*italic*`, `# headers`). Write plain text only — Reddit comments do not render Markdown.

## Resource Index

| Resource | Location | Purpose |
|----------|----------|---------|
| **Post Strategy** | `references/post-strategy.md` | Content angles, anti-AI rules, subreddit cultures, engagement triggers |
| **Comment Strategy** | `references/comment-strategy.md` | (Reserved) Reply patterns, comment-specific tactics |
| **Interaction Patterns** | `references/interaction-patterns.md` | UI automation playbooks (how to click/type/verify) |
| **Sub Archives** | `references/sub-archives.md` | Core info for all subreddits (one file) |
| **Persona Facts** | `PERSONA.md` (skill directory) | Authentic personal facts to use in content |

---

## Subreddit Name Index

### Pre-defined Archives (in `post-strategy.md`)

These subreddits have curated cultural profiles:

| Subreddit | Focus | Posting Bar |
|-----------|-------|-------------|
| `r/Startup_Ideas` | Idea validation, feedback | Medium — show research |
| `r/SaaS` | SaaS building, metrics, growth | Medium-High — data expected |
| `r/TheFounders` | Founder experiences, support | Medium — authenticity valued |
| `r/openclaw` | OpenClaw tool discussions | Low-Medium — technical |

### Dynamic Archive System

For subreddits NOT listed above:

1. **Check for existing archive:** Read `references/sub-archives.md`, find the sub section
2. **If not found:** Execute `interaction-patterns.md` §0 (Dynamic Subreddit Analysis Framework)
3. **Append to file:** Add new sub info to `references/sub-archives.md` using the template at bottom
4. **Use for content generation:** Treat like pre-defined profile

### Supported Subreddit Categories

| Category | Example Subs | Strategy |
|----------|--------------|----------|
| **Startup/Business** | r/Entrepreneur, r/smallbusiness, r/marketing | Use r/Startup_Ideas baseline |
| **Technology** | r/technology, r/programming, r/webdev | Use r/openclaw baseline |
| **AI/ML** | r/artificial, r/MachineLearning, r/LocalLLaMA | Technical + data-driven tone |
| **Productivity** | r/productivity, r/Notion, r/Obsidian | Workflow-focused, practical |
| **General Discussion** | r/CasualConversation, r/AskReddit | Casual, low bar |

**Rule:** If no archive exists in `sub-archives.md` and category match is unclear → always run dynamic analysis (§0) before posting.

---

## Workflow Router

### Create Post

**Trigger:** user asks to publish in a subreddit.

**Content Generation Flow:**

1. **Determine Input Type**
   - If user provides full content (title + body) → use as-is, skip strategy
   - If user provides fuzzy request (e.g., "post about OpenClaw") → execute strategy flow below

2. **Strategy Flow** (for fuzzy requests)
   a. **Identify Target Subreddit** — extract from user request or ask
   b. **Load Sub Profile** — read `references/post-strategy.md` §1 for subreddit rules/tone
   c. **Select Content Angle** — read `references/post-strategy.md` §3 for angle matching intent
   d. **Load Persona Facts** — read `PERSONA.md` (skill directory) for authentic personal facts
   e. **Apply Anti-AI Rules** — read `references/post-strategy.md` §2 to avoid AI-sounding language
   f. **Draft Content** — combine sub profile + angle + persona facts + anti-AI rules
   g. **Add Engagement Hook** — read `references/post-strategy.md` §4 for comment triggers
   h. **Craft Title** — read `references/post-strategy.md` §1 for subreddit-specific title patterns

3. **Confirmation**
   - Echo generated content to user for approval (unless user pre-authorized immediate send)

4. **Publish**
   - Execute `references/interaction-patterns.md` §1 (Create Post workflow)

5. **Log Usage**
   - Update `PERSONA.md` (skill directory) "Used Content Log" table to prevent future contradictions

**Flow Reference:** `references/interaction-patterns.md` §1

---

### Create Comment

**Trigger:** user asks to reply to a post or comment.

**Content Generation Flow:**

1. **Determine Input Type**
   - If user provides full comment text → use as-is
   - If user provides fuzzy request (e.g., "comment on this post") → execute strategy flow below

2. **Strategy Flow** (for fuzzy requests)
   a. **Read Target Post/Comment** — understand context and existing discussion
   b. **Check Subreddit Info**
      - Read `references/sub-archives.md` for this sub
      - **If NOT found:** Execute `interaction-patterns.md` §0 (Dynamic Subreddit Analysis Framework), then append sub info to `references/sub-archives.md`
      - **If found:** Use existing info (tone, rules, restrictions)
   c. **Load Comment Strategy** — read `references/comment-strategy.md` (when available) or apply `post-strategy.md` Human-First rules
   d. **Load Persona Facts** — read `PERSONA.md` (skill directory) for authentic personal facts
   e. **Apply Anti-AI Rules** — read `references/post-strategy.md` (Word Razor, Two-Beat Flow, etc.)
   f. **Draft Comment** — ensure information increment, not just "+1"; match sub tone from step b

3. **Confirmation**
   - Echo generated comment to user for approval (unless pre-authorized)

4. **Publish**
   - Execute `references/interaction-patterns.md` §2 (Create Comment workflow)

5. **Log Usage**
   - Update `PERSONA.md` (skill directory) "Used Content Log" table if persona facts were used

**Flow Reference:** `references/interaction-patterns.md` §2

---

### Upvote

**Trigger:** user asks to like/upvote a post or comment.

**Flow:** Execute `references/interaction-patterns.md` §3 (Upvote workflow)

**No content generation needed.**

## Universal Reliability Loop

For each action step:

1. Snapshot current tab with aria refs.
2. Resolve target candidates by semantic cues (role, accessible name, nearby headings/text).
3. Score candidates (exact intent match > synonym match > position-based guess).
4. Execute one action.
5. Verify outcome with independent evidence (state change, toast, button state, new content visible).
6. If verification fails, rollback one step (or refresh context) and retry with next candidate.

Stop after 3 failed attempts and return a clear diagnostic with:
- last confirmed state
- failing intent
- top 2 alternate hypotheses

## Semantic Targeting Standard

Prefer these signals (highest to lowest confidence):

1. Accessible role + accessible name exact match (e.g., button "Post", "Comment", "Upvote").
2. Accessible name synonym match (e.g., "Submit", "Reply", "Send").
3. Relative context anchor (inside composer region; under post action row; near subreddit title).
4. Visibility and enabled state checks.

Never rely on:
- hardcoded CSS selectors
- hardcoded DOM ids/classes
- absolute XPath
- static child index paths



## Submission Safety Gates

Before clicking final submit:

1. Echo target summary: subreddit/post URL/action/content preview.
2. Ensure required fields are non-empty and within Reddit limits.
3. Confirm account/session is authenticated.
4. Confirm no blocking validation/errors are visible.

If explicit user request already includes immediate-send intent, submit directly.
Otherwise ask one short confirmation.

## Output Contract

After completion, return:

- Action performed
- Target (subreddit/post/comment)
- Result (`success` | `partial` | `failed`)
- Evidence (what changed in UI)
- If failed: next best recovery step

## Fast Troubleshooting

- Login wall/captcha: pause and ask user to complete challenge, then resume from latest snapshot.
- Button exists but disabled: inspect missing required fields or rate limits.
- Multiple matching buttons: choose candidate within nearest semantic container (composer/action row).
- UI redesign: rely on role/name/context fallback chain from `references/interaction-patterns.md`.
