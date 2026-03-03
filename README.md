# XAF DevExpress Agent Skill

An open-source skill set for AI coding assistants that support the [Agent Skills](https://skills.sh) format. Provides practical, code-first guidance for building business applications with [DevExpress XAF (eXpressApp Framework)](https://docs.devexpress.com/eXpressAppFramework/112670/expressapp-framework) — covering both **v24.2** and **v25.1**, on **Blazor Server**, **WinForms**, and **Web API (OData)** platforms.

18 focused skill files — each concise and token-efficient, with source links for deeper exploration.

---

## Installation

### Option A: skills.sh CLI (recommended — works with 40+ agents)

The `npx skills` CLI installs to the correct location for each tool automatically.

**Install all 18 skills globally:**
```bash
npx skills add kashiash/xaf-skills
```

**Install for a specific tool:**
```bash
npx skills add kashiash/xaf-skills -a claude-code
npx skills add kashiash/xaf-skills -a cursor
npx skills add kashiash/xaf-skills -a windsurf
npx skills add kashiash/xaf-skills -a github-copilot
npx skills add kashiash/xaf-skills -a cline
```

**Install a single skill only:**
```bash
npx skills add kashiash/xaf-skills --skill xaf-controllers
```

**Preview what will be installed:**
```bash
npx skills add kashiash/xaf-skills --list
```

#### Where skills are stored per tool

| Tool | Global | Project |
|------|--------|---------|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Cursor | `~/.cursor/skills/` | `.agents/skills/` |
| Windsurf | `~/.codeium/windsurf/skills/` | `.windsurf/skills/` |
| GitHub Copilot | `~/.copilot/skills/` | `.agents/skills/` |
| Cline | `~/.agents/skills/` | `.agents/skills/` |

---

### Option B: Claude Code plugin marketplace

Run each command **separately** and wait for it to complete:

```shell
/plugin marketplace add kashiash/xaf-skills
```
```shell
/plugin install xaf-devexpress@xaf-devexpress-skills
```

### Option C: Team / project-level — Claude Code (shared settings.json)

Add to your project's `.claude/settings.json` to share with the whole team automatically:

```json
{
  "extraKnownMarketplaces": {
    "xaf-devexpress-skills": {
      "source": {
        "source": "github",
        "repo": "kashiash/xaf-skills"
      }
    }
  },
  "enabledPlugins": {
    "xaf-devexpress@xaf-devexpress-skills": true
  }
}
```

### Option D: Manual

```bash
git clone https://github.com/kashiash/xaf-skills
# Claude Code
cp -r xaf-skills/xaf* ~/.claude/skills/
# Cursor
cp -r xaf-skills/xaf* ~/.cursor/skills/
# Windsurf
cp -r xaf-skills/xaf* ~/.codeium/windsurf/skills/
```

---

## What This Skill Covers

Once installed, the AI agent can assist with the full XAF development workflow:

- **Data modeling** — XPO persistent objects and EF Core entities, associations, migrations, optimistic locking
- **Controllers & Actions** — SimpleAction, PopupWindowShowAction, DialogController, chaining popups, async/await in Blazor, view refresh patterns
- **UI customization** — Built-in editors, custom property editors (Blazor & WinForms), custom list editors
- **NonPersistent objects** — Custom data sources, API wrappers, input dialogs without database storage
- **Security** — Role-based access control, Standard/AD/OAuth2 authentication, programmatic permission checks, JWT
- **Multi-tenancy** — Separate database per tenant, ITenantProvider, custom resolvers (v24.2+)
- **Web API** — OData v4 endpoints, JWT auth, custom actions, `$filter`/`$expand`/`$select`
- **Validation** — All built-in rule types, custom rules, programmatic validation
- **Reports** — XtraReports setup, predefined reports, programmatic export (PDF/XLSX/DOCX)
- **Dashboards** — Dashboard analytics module, custom storage, data sources
- **Office modules** — File attachments, Spreadsheet editor, RichText editor, PDF viewer
- **Platform-specific** — Blazor thread safety (InvokeAsync), WinForms XtraGrid access, background workers
- **Conditional Appearance** — `[Appearance]` attribute, hide/disable/color UI elements by criteria
- **Deployment** — IIS/Azure/Docker, ClickOnce/MSI, DevExpress license, Serilog, health checks

---

## Target Audience

- Teams building line-of-business applications with DevExpress XAF
- Developers new to XAF who need quick reference patterns
- Experienced XAF developers exploring less common features (NonPersistent, custom editors, multi-tenancy)
- Teams setting up AI-assisted code reviews and generation for XAF projects

---

## Usage Examples

After installation, invoke the skills in your AI agent:

```
Use the xaf-controllers skill to create a PopupWindowShowAction that opens a selection dialog.
```

```
Use the xaf-ef-models skill to define a one-to-many relationship between Order and OrderLine.
```

```
Use the xaf-security skill to set up role-based permissions with a deny-all-by-default policy.
```

```
Use the xaf-nonpersistent skill to build a custom data source for an external REST API.
```

Agents supporting auto-discovery will load the relevant skill automatically based on task context.

---

## Available Skills

| Skill | Description |
|-------|-------------|
| `xaf` | Master index — routes to the right sub-skill for any XAF topic |
| `xaf-xpo-models` | XPO persistent objects, base classes, attributes, associations, PersistentAlias |
| `xaf-ef-models` | EF Core entities, BaseObject, DbContext, IDbContextFactory, migrations |
| `xaf-controllers` | Controllers, all Action types, DialogController, chained popups, ObjectChanged/CurrentObjectChanged refresh |
| `xaf-editors` | Built-in property editors, list editors, GridListEditor/DxGridListEditor customization |
| `xaf-custom-editors` | Custom property editors (Blazor & WinForms), custom list editors, registration |
| `xaf-nonpersistent` | NonPersistent objects, ObjectsGetting, DynamicCollection, popup input forms |
| `xaf-security` | Security system, roles, permissions, OAuth2, JWT, programmatic permission checks |
| `xaf-multi-tenant` | Multi-tenancy, ITenantProvider, per-tenant DB isolation (v24.2+) |
| `xaf-web-api` | OData Web API, JWT auth, custom endpoints, $filter/$expand |
| `xaf-validation` | Validation rules, RuleCriteria, custom RuleBase, programmatic validation |
| `xaf-reports` | XtraReports v2, predefined reports, programmatic export, designer |
| `xaf-dashboards` | Dashboard module, custom storage, data sources, permissions |
| `xaf-office` | File attachments, Spreadsheet, RichText, PDF viewer |
| `xaf-blazor-ui` | Blazor Server UI, InvokeAsync thread safety, custom Razor ViewItems, JS interop |
| `xaf-winforms-ui` | WinForms UI, XtraGrid access, background workers, splash screen |
| `xaf-conditional-appearance` | [Appearance] attribute, criteria expressions, dynamic appearance from code |
| `xaf-deployment` | IIS/Azure/Docker, ClickOnce/MSI, DevExpress license, Serilog, health checks |

---

## DevExpress MCP Server

For even deeper, real-time documentation access during AI sessions, add the DevExpress MCP server to `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "dxdocs": {
      "type": "http",
      "url": "https://api.devexpress.com/mcp/docs"
    },
    "dxdocs-v251": {
      "type": "http",
      "url": "https://api.devexpress.com/mcp/docs/v25.1"
    }
  }
}
```

This gives the AI agent live access to the full DevExpress documentation API during code generation.

---

## Skill Structure

Each skill is a self-contained folder with a single `SKILL.md` file:

```
xaf-skills/
├── .claude-plugin/
│   └── marketplace.json
├── xaf/SKILL.md                      ← master index
├── xaf-controllers/SKILL.md
├── xaf-ef-models/SKILL.md
├── xaf-xpo-models/SKILL.md
└── ... (18 skill folders total)
```

Each `SKILL.md` follows the [Agent Skills format](https://skills.sh/docs):

```yaml
---
name: xaf-topic-name
description: One-line description for agent skill selection (max 1024 chars)
license: MIT
compatibility: opencode, claude-code
---

# XAF: Topic Name

## Section
...code examples, tables, patterns, source links...
```

---

## Contributing

PRs welcome. When adding or updating a skill:

1. Keep each SKILL.md focused on one topic — short, code-first, with source links
2. Include a table of contents or section headers for fast scanning
3. Add links to official docs at the bottom of each file
4. Test that the skill name in frontmatter matches the folder name

---

## Related Resources

- [XAF Documentation](https://docs.devexpress.com/eXpressAppFramework/112670/expressapp-framework)
- [XAF Getting Started (Blazor)](https://docs.devexpress.com/eXpressAppFramework/402189/getting-started/in-depth-tutorial-blazor)
- [XAF Getting Started (WinForms)](https://docs.devexpress.com/eXpressAppFramework/402684/getting-started/in-depth-tutorial-winforms-webforms)
- [DevExpress MCP Server](https://community.devexpress.com/Blogs/news/archive/2025/10/16/transform-your-development-experience-with-the-devexpress-mcp-server.aspx)
- [DevExpress GitHub Examples](https://github.com/DevExpress-Examples)
- [DevExpress Support Center](https://supportcenter.devexpress.com)
- [Agent Skills Directory (skills.sh)](https://skills.sh)

---

## License

MIT — free to use, modify, and distribute.
