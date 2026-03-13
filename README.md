<h1 align="center">Claude Figma Skills</h1>
<p align="center">A collection of Claude Code skills for Figma — review designs, act on comments, sync design tokens and more.</p>
<p align="center"><code>claude-code</code> · <code>claude-skill</code> · <code>figma</code> · <code>design-tokens</code></p>
<p align="center">Requires <a href="https://docs.anthropic.com/en/docs/claude-code">Claude Code</a> and <a href="https://www.figma.com/downloads/">Figma Desktop</a> (any plan)</p>

## Skills

| Skill | What it does | How to use |
|---|---|---|
| **figma-review** | Claude reviews your design and posts actionable feedback as comments pinned to specific elements in Figma. Analyzes hierarchy, spacing, color, and consistency. | `/figma-review` + Figma URL |
| **figma-autopilot** | You leave comments in Figma describing what to change, Claude reads them, executes the design changes, replies to each comment, and resolves them when done. | `/figma-autopilot` + Figma URL |
| **figma-fix** | Runs figma-review then figma-autopilot in sequence — Claude reviews the design, presents a plan, and immediately fixes what it finds. Takes a verification screenshot at the end. | `/figma-fix` + Figma URL |
| **figma-extract** | Claude exports design tokens (colors, spacing, typography, radius), assets, and component specs from Figma into structured code files (CSS, Tailwind, JSON, or markdown). | `/figma-extract` (select element in Figma first) |
| **figma-import** | Claude scans your codebase for design tokens (colors, spacing, typography, radius, opacity) and pushes them to Figma as organized variable collections. Can also generate a visual documentation page. | `/figma-import` (run from project directory) |
| **figma-tickets** | Claude reads all comment threads on a Figma page, categorizes them (feedback, question, resolved), and generates structured design tickets saved as a markdown file to your Desktop. | `/figma-tickets` + Figma URL |

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

Each skill is a single `SKILL.md` file inside its own folder. Claude Code picks them up automatically — no restart needed.

### 2. Set up dependencies

| Dependency | Required by |
|---|---|
| **figma-console MCP** | figma-fix, figma-autopilot, figma-extract, figma-import, figma-tickets |
| **Figma REST API token** | figma-review, figma-fix, figma-autopilot |
| **Python 3** | figma-review, figma-fix, figma-autopilot, figma-tickets |

**figma-console MCP** — gives Claude Code read + write access to your Figma files via `localhost:9222`. Works on any Figma plan (free, Starter, Pro) — no Enterprise required.

1. Install the [Figma Console MCP](https://github.com/southleft/figma-console-mcp) plugin in Figma Desktop
2. Open the Figma plugin (Plugins > Development > Figma Console MCP)
3. Add the MCP server to your `~/.claude.json` following the [setup instructions](https://github.com/southleft/figma-console-mcp#setup)

**Figma REST API token** — used for reading/posting comments and fetching screenshots:

```bash
# Add to your ~/.zshrc or ~/.bashrc
export FIGMA_ACCESS_TOKEN="your-token-here"
```

Generate a token at: Settings > Security > Personal access tokens in Figma.

**Python 3** — used for batch operations (posting comments, parsing JSON). Standard library only, no pip packages. Included with macOS. On Linux: `sudo apt install python3`.## License

[MIT](LICENSE)

---

Made by [santiagoalonso.com](https://santiagoalonso.com)
