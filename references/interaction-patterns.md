# Reddit Interaction Patterns (Accessibility-First)

## 0) Dynamic Subreddit Analysis Framework

**Purpose:** Handle new/unseen subreddits that don't have pre-defined cultural profiles in `post-strategy.md`.

### When to Use

Trigger this framework when:
- User requests posting/commenting in a subreddit NOT listed in `post-strategy.md` §1
- Subreddit name is extracted from user request (e.g., "post to r/Entrepreneur")
- No cached archive exists in `sub-archives/` directory

---

## Phase 1: Fetch ALL Data via Reddit API (Fast Path)

**Use Reddit's free JSON API** — no auth required, ~60 req/min limit. **Rules ARE included in API response!**

### Step 1: Call Reddit API

```bash
curl -sL -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)" "https://www.reddit.com/r/<subname>/about.json"
```

**⚠️ Important:** Must include proper User-Agent header, or Reddit will block with "network policy" error.

**Expected fields from API:**

| Field | JSON Path | Example |
|-------|-----------|---------|
| **Subreddit name** | `.display_name` | `Entrepreneur` |
| **Title** | `.title` | `Entrepreneur` |
| **Description** | `.public_description` | Short blurb |
| **Full rules + description** | `.description` | **Complete rules in Markdown/HTML** |
| **Members** | `.subscribers` | `5106958` |
| **Active users** | `.accounts_active` | `12345` |
| **Created (UTC)** | `.created_utc` | `1219348682.0` |
| **Subreddit type** | `.subreddit_type` | `public` |
| **NSFW** | `.over18` | `false` |
| **Submit text** | `.submit_text` | Posting requirements |
| **Submission type** | `.submission_type` | `self`, `link`, `any` |

### Step 2: Parse and Validate

**Success criteria:**
- `.subreddit_type` is `public` (not `private`, `restricted`, `gold_restricted`)
- `.subscribers` > 0
- No error in response

**Error cases:**

| Response | Meaning | Action |
|----------|---------|--------|
| `{"message": "Not Found"}` | Subreddit doesn't exist | Check spelling, suggest alternates |
| `{"message": "private"}` | Private subreddit | Report "Subreddit is private", ask for alternate |
| `{"message": "quarantined"}` | Quarantined sub | Warn user, may need opt-in |
| HTML "Blocked" page | Missing/bad User-Agent | Retry with proper UA header |

### Step 3: Extract Rules from `.description` Field

The `.description` field contains full rules in Markdown format. Example parsing:

```bash
# Extract description (contains rules)
curl -sL -A "Mozilla/5.0" "https://www.reddit.com/r/<subname>/about.json" \
  | jq -r '.data.description'
```

**Parse rules from Markdown:**
```markdown
##Submission/commenting Rules:
 
1) **10 comment karma in /r/Entrepreneur to post**  
   [rule details...]

2) **No Promotion**  
   [rule details...]

3) **No Personal Attacks**  
   [rule details...]
```

Extract top 3-5 rules by finding numbered patterns: `^\d+\)\s*\*\*(.+?)\*\*`

### Step 4: Generate Complete Profile

```markdown
## r/{SUBNAME}

| Field | Value |
|-------|-------|
| **Members** | {subscribers} |
| **Active Users** | {accounts_active} |
| **Description** | {public_description} |
| **Created** | {date from created_utc} |
| **Submission Type** | {submission_type} |
| **Post Requirements** | {submit_text} |

### Top Rules
1. {Rule 1 title}: {summary}
2. {Rule 2 title}: {summary}
3. {Rule 3 title}: {summary}
4. {Rule 4 title}: {summary}
5. {Rule 5 title}: {summary}

### Notes
- All data fetched via API (no browser needed)
- Full rules available in `.description` field
```

---

## Phase 2: Browser Fallback (Only if API Fails)

**Use browser only when:**
- API returns error (private/quarantined/blocked)
- `.description` field is empty or malformed
- Need to analyze recent post patterns for tone/content strategy

### Step 5: Analyze Recent Posts (Optional, Browser)

```
URL: https://www.reddit.com/r/<subname>/new
```

Navigate to /new, snapshot, analyze:

| Signal | What to Look For |
|--------|------------------|
| **Post frequency** | Timestamp of last 5-10 posts |
| **Common flairs** | Flair text near post titles |
| **Title patterns** | Question vs statement ratio |
| **Engagement** | Comment counts, upvote ratios |
| **Content type** | Text vs links vs images |

---

## Phase 3: Cache to Archive

### Step 6: Append to `sub-archives.md`

```markdown
## r/{SUBNAME}

| Field | Value |
|-------|-------|
| **Members** | X |
| **Posting Threshold** | {from submit_text} |
| **AI Detection** | {check rules for "AI" keyword} |
| **Language** | {infer from posts or description} |
| **Tone** | {infer from description} |
| **Self-promo** | {check rules} |

### Top 3 Rules
1.
2.
3.

### What Works
-

### Title Patterns
-

### Notes

```

---

## Integration with Content Generation

After generating the dynamic archive:

1. **Load the archive** as if it were a pre-defined profile in `post-strategy.md` §1
2. **Extract cultural signals** (tone, welcome/avoid patterns, title styles)
3. **Apply to content generation** following `SKILL.md` Workflow Router
4. **Flag for user review** if confidence is low

### Confidence Scoring

| Level | Criteria | Action |
|-------|----------|--------|
| **High** | API success + rules parsed + metadata complete | Proceed |
| **Medium** | API success but rules sparse | Generate, flag for review |
| **Low** | API failed or description empty | Use browser fallback |

### Error Handling

| Error | Recovery |
|-------|----------|
| API returns "private" | Report "Subreddit is private", ask for alternate |
| API returns "Not Found" | Check spelling, suggest similar subs |
| HTML "Blocked" response | Retry with different User-Agent |
| `.description` empty | Proceed without rules, flag as "unmoderated or hidden" |
| Rate limited | Wait 60s and retry, or use cached data |

---

## Quick Reference: API Endpoints

```bash
# Subreddit metadata + rules
curl -sL -A "Mozilla/5.0" "https://www.reddit.com/r/{sub}/about.json"

# Recent posts (for content analysis)
curl -sL -A "Mozilla/5.0" "https://www.reddit.com/r/{sub}/new.json?limit=10"

# Hot posts
curl -sL -A "Mozilla/5.0" "https://www.reddit.com/r/{sub}/hot.json?limit=10"

# Top posts (all time)
curl -sL -A "Mozilla/5.0" "https://www.reddit.com/r/{sub}/top.json?limit=10"
```

**All endpoints support `.json` suffix for API access.**

---

## 1) Create Post

### Strategy Layer (Content Generation)

**Before executing this playbook:**

- If user provided fuzzy request (e.g., "post about OpenClaw") → execute content generation flow in `SKILL.md` Workflow Router first
- Load `references/post-strategy.md` for:
  - Subreddit cultural profile (§1)
  - Anti-AI writing rules (§2)
  - Content angle selection (§3)
  - Engagement triggers (§4)
- Load `PERSONA.md` for authentic personal facts
- Generate title and body content
- Obtain user confirmation (unless pre-authorized)

**This playbook assumes content is ready to publish.**

---

### Preconditions
- User provides: subreddit + title + body (or content generated via strategy layer)
- Browser is open on reddit and authenticated
- Content confirmed by user (if not pre-authorized)

### Steps
1. Navigate to subreddit URL (`/r/<name>`).
2. Snapshot (`refs=aria`).
3. Find post composer entry by semantic labels like:
   - "Create post"
   - "Create"
   - "Post"
4. Open composer and re-snapshot.
5. Fill title field (role textbox, name includes "Title").
6. Fill body field when provided (textbox/editor region named "Body", "Text", or "Post body").
7. Optional type switch (Text/Image/Link/Poll) by tab/button role with matching names.
8. Verify submit button semantic name in {"Post", "Submit", "Publish"} and enabled.
9. Click submit.
10. Verify success via one or more:
   - Post detail page opens
   - New post title visible
   - Success toast/banner

### Failure Recovery
- If composer not found: search global "Create" button from top nav.
- If subreddit posting restricted: return restriction reason and ask alternate subreddit.

---

## 2) Create Comment

### Strategy Layer (Content Generation)

**Before executing this playbook:**

- If user provided fuzzy request (e.g., "comment on this post") → execute content generation flow in `SKILL.md` Workflow Router first
- Load `references/comment-strategy.md` for comment-specific tactics (when available)
- Load `references/post-strategy.md` §2 for anti-AI writing rules (shared)
- Load `PERSONA.md` for authentic personal facts
- Read target post/comment to understand context
- Generate comment content with information increment (not just "+1")
- Obtain user confirmation (unless pre-authorized)

**This playbook assumes content is ready to publish.**

---

### Preconditions
- User provides: post URL + comment text (or content generated via strategy layer)
- Browser is open on reddit and authenticated
- Content confirmed by user (if not pre-authorized)

### Steps
1. Open target post URL.
2. Snapshot (`refs=aria`, depth=10) to find initial placeholder textbox.
3. Locate placeholder textbox (typically `textbox [ref=e267]` with `/placeholder: Join the conversation`).
   - ⚠️ Do NOT type into this placeholder ref — it is a custom web component (`faceplate-textarea-input`), not a real textarea.
4. **Click** the placeholder textbox ref to activate the Slate editor.
5. **Re-snapshot at depth=13** — this is required. The active editor appears deeply nested inside the ad block's thumbnail link element in the ARIA tree (an Reddit layout quirk). At shallower depths it won't appear.
6. Find `textbox [active] [ref=eXXXX]` — the high-numbered ref is the real Slate editor. Also note `button "Comment" [ref=eYYYY]` at the same level — you'll need it for submit.
7. Type comment text into the **active** textbox ref.
8. Click the `button "Comment"` ref found in step 6 (do NOT use evaluate to click "Comment" — use the direct ref to avoid hitting wrong elements).
9. Verify: evaluate `document.body.innerText.includes('<unique snippet>')`.

### ARIA Tree Quirk — Where the Editor Lives
After clicking the placeholder, Reddit renders the full Slate editor nested inside:
```
generic [ad block] [ref=e205]:
  link [Thumbnail image: ...] [ref=e253]:
    paragraph [ref=eXXX]
    textbox [active] [ref=eXXXX]   ← your typing target
      paragraph [ref=eXXX]
      button "Comment" [ref=eYYYY]  ← your submit target
      button "Cancel" [ref=eZZZZ]
    button "Show formatting options" [ref=...]
```
This structure is consistent across posts — always depth=13 to see it.

### Failure Recovery
- If locked thread/mod restrictions: report exact restriction text.
- If submit disabled: ensure non-empty content and no markdown-mode validation blocks.
- If "The field is required and cannot be empty" appears: you clicked submit on an empty editor. Reload the page and restart from step 2.
- If comment not found after submit: page may have redirected to user profile (normal Reddit behavior after submit) — navigate back to post and verify with evaluate.

---

## 3) Upvote

### Preconditions
- User provides target URL (post/comment) or clear target description.

### Steps
1. Open target context.
2. Snapshot (`refs=aria`).
3. Locate vote control near intended entity (post card or comment container).
4. Pick control whose accessible name implies upvote semantics:
   - "Upvote"
   - "Vote up"
   - localized equivalent with up-arrow context
5. Click once.
6. Verify state change via one or more:
   - button pressed/selected state
   - score increment (if visible)
   - control style/state indicates active vote

### Failure Recovery
- If already upvoted, return idempotent success.
- If ambiguous (multiple upvotes), anchor to nearest container that matches target title/snippet.

---

## 4) Candidate Scoring Heuristic

Use weighted scoring when multiple elements match.

- +8 exact role match
- +10 exact/strong accessible-name intent match
- +6 synonym match
- +7 in expected semantic container (composer/action row/comment item)
- +5 visible in viewport
- +2 enabled
- -3 outside viewport and no scroll evidence
- -5 conflicts with target entity context

Pick highest score above threshold 10.

If top-2 delta < 4, mark as ambiguous and do not click yet.

## 4.1) Uniqueness + Context-Lift Fallback

When selector returns multiple nodes:

1. Stop action.
2. Lift context to nearest parent semantic region.
3. Re-score with composite conditions:
   - role
   - aria-label / placeholder / accessible text
   - parent region match
4. If still ambiguous, return top 3–5 candidates and let planner choose.
5. Never spin on the same ambiguous ref id.

This avoids deadlocks caused by volatile refs like `e47` that map to multiple nodes.

---

## 5) Synonym Bank (UI labels)

- Post submit: Post / Submit / Publish
- Comment submit: Comment / Reply / Post
- Upvote: Upvote / Vote up / Like
- Composer: Create post / Start a post

---

## 6) Anti-Brittle Rules

- Do not bind to `id`, dynamic class hashes, or nth-child paths.
- Do not assume fixed layout for old/new Reddit.
- Always refresh semantic map after action-induced rerender.
- Prefer intent anchor + local container instead of global first match.

---

## 7) Known Pitfalls (Hard-Won)

| Pitfall | Cause | Fix |
|---------|-------|-----|
| Typed text disappears / textbox stays empty | Reddit uses React — raw DOM `.value` JS assignment is silently ignored | Always use CDP `act type` (not `evaluate` to set `.value`) |
| `faceplate-textarea-input` doesn't accept input | It's a custom web component shell, not a real textarea | Click it first to expand the real Slate editor, then re-snapshot and target the new `textbox [active]` ref |
| "The field is required" error on submit | Clicked Comment button while textbox was still empty (typed into wrong ref) | Reload page, restart from step 1, be sure to click placeholder → snapshot → find `[active]` ref before typing |
| Comment not visible after clicking Comment | Reddit redirected to user profile (`/user/szy1840/`) after submit — this is normal | Navigate back to the post URL and verify with `evaluate document.body.innerText.includes(...)` |
| `textbox [active]` not visible in snapshot | Snapshot depth too shallow (< 13) | Always use `depth=13` after activating the editor |
| Wrong element clicked when using evaluate | `querySelectorAll('button')` may match multiple "Comment" buttons on the page | Use the direct `ref` from snapshot, not evaluate-based queries, for submit |
| Locked post — no comment box | Some subreddit posts are locked by mods | Check ARIA tree for "Locked post" text before attempting; skip and move to next post |
