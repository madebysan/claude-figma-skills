<h1 align="center">Claude Figma Skills</h1>
<p align="center">A Claude Code skill bundle for Figma. Review designs, execute comments, sync tokens, and more.<br>
Moves mechanical design-review work out of the queue so you can focus on judgment calls.</p>
<p align="center"><code>claude-code</code> · <code>claude-skill</code> · <code>figma</code> · <code>design-tokens</code></p>
<p align="center">Requires <a href="https://docs.anthropic.com/en/docs/claude-code">Claude Code</a> and <a href="https://www.figma.com/downloads/">Figma Desktop</a> (any plan)</p>

https://github.com/user-attachments/assets/6ee9c093-38f4-4b40-90a5-0c3ca05e6726

<p align="center"><sub><code>/figma-review</code> in action. One of eight skills in this kit.</sub></p>

---

A collection of Claude skills that act like an AI design intern in Figma. Someone who does the busy, time-consuming work and doesn't just live in Claude but also works like any other designer, directly in Figma, making designs, editing them and leaving comments (plus modify nodes, export tokens, import tokens, see the actual canvas via screenshots, run reviews).

It's for designers and design-engineers already using Claude Code. The underlying MCP works on any Figma tier, including the free one.

## Skills

| Skill | What it does | How to use |
|---|---|---|
| **figma-review** | Claude reviews your design and posts actionable feedback as comments pinned to specific elements. Covers hierarchy, spacing, color, consistency. | `/figma-review` + Figma URL |
| **figma-autopilot** | You leave comments describing changes, Claude reads them, executes the design changes, replies, and resolves each thread. | `/figma-autopilot` + Figma URL |
| **figma-fix** | Runs figma-review, presents a plan, then immediately runs figma-autopilot on what it finds. Takes a verification screenshot at the end. | `/figma-fix` + Figma URL |
| **figma-extract** | Exports design tokens (colors, spacing, typography, radius), assets, and component specs into structured code files (CSS, Tailwind, JSON, or markdown). | `/figma-extract` (select element in Figma first) |
| **figma-import** | Scans your codebase for design tokens (colors, spacing, typography, radius, opacity) and pushes them to Figma as organized variable collections. Optional visual documentation page. | `/figma-import` (run from project directory) |
| **figma-tickets** | Reads all comment threads on a Figma page, categorizes them (feedback, question, resolved), and generates structured design tickets as a markdown file on your Desktop. | `/figma-tickets` + Figma URL |
| **figma-file-cleanup** | Audits a Figma file for orphan library refs, unbound colors, detached text styles, dark-mode inconsistencies, and naming issues. Rebinds to your design system tokens, captures before/after screenshots, and writes a final cleanup report. | `/figma-file-cleanup` + Figma URL |
| **figma-component-doc** | Builds a structured doc frame next to a selected component (or component set) with description, variants (visual + intent), anatomy of composite parts, properties with defaults, and usage notes. Auto-discovers the file's own text styles and color variables. | `/figma-component-doc` (select component in Figma first) |

## How I use these

Reach for any of these whenever the work calls for it. They aren't on a schedule.

`figma-review` is more of a proof of concept than a daily driver. It shows that Claude Code can leave comments inside Figma, pinned to specific elements. Fork it and swap the reviewer persona to specialize the skill: a UX copy reviewer, an accessibility auditor, a design system enforcer. Whatever review role you want, the skill is the template.

`figma-autopilot` handles design changes left as comments, the "move 4px right" or "use token X" kind. Drains the queue without interrupting me.

`figma-extract` at hand-off, to pull the final tokens out of the file and into the codebase. `figma-import` later when the codebase has drifted and the source-of-truth copy needs pushing back to Figma.

`figma-tickets` converts comment threads into a structured backlog a PM can actually file.

`figma-file-cleanup` is for the periodic deep clean. When a file has accumulated orphan tokens from defunct libraries, broken dark mode, or generic layer names, this is the one to reach for. Useful before handoffs and design system migrations.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [Figma Desktop](https://www.figma.com/downloads/) (any plan: free, Starter, Pro, Enterprise all work)
- Python 3 (included with macOS; `sudo apt install python3` on Linux)
- A Figma personal access token for the REST-API skills
- The [figma-console MCP](https://github.com/southleft/figma-console-mcp) plugin for skills that modify Figma content

## Install

### 1. Copy the skills

```bash
# Clone the repo
git clone https://github.com/madebysan/claude-figma-skills.git

# Copy all skills
cp -r claude-figma-skills/skills/* ~/.claude/skills/

# Or copy individual skills
cp -r claude-figma-skills/skills/figma-review ~/.claude/skills/
cp -r claude-figma-skills/skills/figma-import ~/.claude/skills/
```

Each skill is a single `SKILL.md` file inside its own folder. Claude Code picks them up automatically. No restart needed.

### 2. Set up dependencies

| Dependency | Required by |
|---|---|
| **figma-console MCP** | figma-fix, figma-autopilot, figma-extract, figma-import, figma-tickets, figma-file-cleanup, figma-component-doc |
| **Figma REST API token** | figma-review, figma-fix, figma-autopilot |
| **Python 3** | figma-review, figma-fix, figma-autopilot, figma-tickets |

**figma-console MCP** gives Claude Code read + write access to Figma files via `localhost:9222`. Works on any Figma plan. No Enterprise required.

1. Install the [Figma Console MCP](https://github.com/southleft/figma-console-mcp) plugin in Figma Desktop
2. Open the plugin (Plugins → Development → Figma Console MCP)
3. Add the MCP server to your `~/.claude.json` per the [setup instructions](https://github.com/southleft/figma-console-mcp#setup)

**Figma REST API token.** Used for reading and posting comments and fetching screenshots. Generate at Settings → Security → Personal access tokens.

```bash
# Add to ~/.zshrc or ~/.bashrc
export FIGMA_ACCESS_TOKEN="your-token-here"
```

**Python 3.** Used for batch operations (posting comments, parsing JSON). Standard library only, no pip packages.

## Known issues / caveats

- **figma-console MCP is third-party.** If Figma's internal plugin API changes, the MCP (and any skill that depends on it) may temporarily break until the MCP's maintainer updates. The REST-only skills (figma-review, parts of figma-autopilot) are less exposed.
- **Personal access tokens are poll-based.** REST-API skills that read comments poll the file on demand. There is a short lag between posting a comment and Claude picking it up. This is a Figma API constraint, not a skill one.

## License

[MIT](LICENSE)

---

Made by [santiagoalonso.com](https://santiagoalonso.com)
