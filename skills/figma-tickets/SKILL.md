---
name: figma-tickets
description: Converts Figma page/frame comments into structured design tickets. This skill should be used when the user provides a Figma URL and wants to extract comment threads and turn them into actionable design/UX tickets. Triggers on phrases like "figma tickets", "turn these comments into tickets", "extract tickets from Figma", or when /figma-tickets is invoked with a Figma link.
---

# Figma to Ticket

## Purpose

Extract all comment threads from a Figma page or frame and convert them into structured design/UX tickets, saved as a markdown file on the Desktop. This bridges QA feedback living in Figma comments with an actionable ticket backlog.

## SCOPE CONSTRAINT

Only produce **design and UX tickets**. Output must exclusively cover: screens, flows, UI components, interaction patterns, visual states, motion, copy, and layout.

**NEVER** create tickets for backend logic, APIs, databases, data models, or infra. If eng work is required, add one line in Dependencies: `"Requires: [brief eng need]"`.

**NEVER** modify Figma comments — do not resolve, delete, or edit any comments. This skill is read-only against Figma.

## Workflow

### Step 1: Validate Figma Connection

Before starting, verify the Figma MCP connection:

1. Run `figma_get_status` to check WebSocket connectivity
2. If not connected, instruct the user to open the Desktop Bridge plugin in Figma Desktop (Plugins > Development > Figma Desktop Bridge)
3. Do not proceed until the connection is confirmed

### Step 2: Parse the Figma URL

Extract from the provided Figma URL:
- **fileKey** — the alphanumeric ID in the URL path (e.g., `cXLeGNOjpN5bvXrlHE9aRm`)
- **node-id** — from the `node-id` query parameter (e.g., `7345-17056`)

Use `figma_navigate` to connect to the file if not already connected.

### Step 3: Identify the Target Page/Frame

Use `figma_get_file_data` with the extracted node ID to determine:
- The page/frame name
- Its child node IDs (needed to filter comments)

Collect all node ID prefixes that belong to the target page/frame and its children.

### Step 4: Fetch All Comments

Call `figma_get_comments` with `include_resolved: true` to get every comment in the file.

If the result is too large for context, save to a temp file and process with Python:

1. Parse the JSON comment data
2. Build thread structures — group replies under their parent comment using `parent_id`
3. Filter to only threads pinned to nodes within the target page/frame (match `client_meta.node_id` against the collected node ID prefixes)

For each thread, extract:
- **Thread ID** (`id`) — for constructing Figma links
- **Author** (`user.handle`) — who started the thread
- **Message** (`message`) — the comment text
- **Created date** (`created_at`)
- **Resolved status** (`resolved_at` — null means open)
- **Node ID** (`client_meta.node_id`) — what the comment is pinned to
- **Replies** — all child comments with their author and message

### Step 5: Construct Figma Links

For each comment thread, construct a clickable Figma link to the node the comment is attached to:

```
https://www.figma.com/design/{fileKey}/?node-id={nodeId}
```

Where `nodeId` uses the colon format from `client_meta.node_id` (e.g., `7385:10630`), URL-encoded as `7385-10630` in the link.

### Step 6: Analyze and Categorize Comments

Process each comment thread and classify it:

- **Question** — comment is asking a question (contains `?`, or phrasing like "should we", "do we need", "what is", etc.). Flag with `[QUESTION]` in the ticket.
- **Design feedback** — comment describes a visual or UX issue (alignment, spacing, color, consistency, copy). These become design tickets.
- **Discussion** — multi-reply thread with back-and-forth. Summarize the discussion outcome. If consensus was reached, ticket the agreed action. If not, flag as `[NEEDS DECISION]`.
- **Resolved** — comment was resolved. Include in the output but mark as `[RESOLVED]` so the user can see what's already been addressed.
- **Noise** — emoji-only, single-word acknowledgements ("ok", "."), or non-actionable content. Skip these entirely.

### Step 7: Generate Tickets

Apply the ticket-creator format. Use this exact structure:

```markdown
# [Page/Frame Name] — Design Tickets from Figma Comments

**Source:** [Figma link to the page/frame]
**Extracted:** [current date]
**Total comment threads:** [count]
**Tickets generated:** [count]
**Skipped (noise/non-actionable):** [count]

## Feature Overview
[One paragraph describing the page/frame context — what it contains, what feature or flow it represents. Derived from the page name and the nature of the comments.]

---

## Ticket 1: <Verb-first descriptive title>

**Type:** Design Story | Design Task | Design Spike
**Status:** [OPEN] | [RESOLVED] | [QUESTION] | [NEEDS DECISION]

**Summary:** One sentence. What is being designed or fixed and why.

**Context & Constraints:** Relevant context from the comment thread, including who raised the issue and the discussion.

**Source comment:**
> "Original comment text" — *Author Name, date*
> [View in Figma](figma-link-to-node)

**Replies (if any):**
> "Reply text" — *Reply Author, date*

**Acceptance Criteria (only if non-obvious):** Skip if the deliverable is self-evident.

**Dependencies:** [Other tickets or eng needs]

---

## Ticket 2: ...

---

## Needs Clarification
- [Questions from comments that don't have enough context to become tickets]
- [Unresolved discussions that need a decision before work can begin]

## Skipped Comments
- [Brief list of comment threads that were skipped and why — e.g., "emoji-only", "resolved acknowledgement"]
```

### Step 8: Apply Style Rules

- **Terse, no filler.** Every word earns its place.
- **Titles scannable on a board** — front-load verb and domain.
- **Preserve attribution.** Always include who said what in the source comment block.
- **Questions stay as questions.** Do not invent answers. Flag with `[QUESTION]` and put in Needs Clarification if there's no resolution in the thread.
- **Group related comments.** If multiple comment threads address the same UI element or issue, combine them into a single ticket and cite all source comments.
- **Do NOT assign priority.** Priority is not derivable from Figma comments.
- **Resolved threads get `[RESOLVED]` status** but are still included for completeness.

### Step 9: Export to Desktop

Save the output as a markdown file:
- **Filename:** `[page-name]-figma-tickets.md` (kebab-case)
- **Path:** `~/Desktop/`
- Confirm the filename with the user before saving, or use a sensible default

### Step 10: Confirm with User

After generating, ask:
- Does this capture the right scope?
- Any tickets that should be split further or merged?
- Anything in Needs Clarification you can answer now?

## What NOT to Do

- Do not resolve, delete, or modify any Figma comments
- Do not create backend, API, or infrastructure tickets
- Do not invent tickets for vague or non-actionable comments — send to Needs Clarification
- Do not assign priority levels
- Do not add filler acceptance criteria that restate the title
- Do not skip the Feature Overview
- Do not process comments from outside the specified page/frame
