# XAF DevExpress Skills for Claude Code / OpenCode

A comprehensive AI skill set for [DevExpress XAF (eXpressApp Framework)](https://docs.devexpress.com/eXpressAppFramework/112670/expressapp-framework) — 18 reference files for AI agents and developers.

**Versions covered:** v24.2 and v25.1
**Platforms:** Blazor Server, WinForms, Web API (OData)

## Installation

### Via plugin marketplace (recommended)

Run each command separately and wait for it to complete before running the next:

**Step 1** — add the marketplace:
```shell
/plugin marketplace add kashiash/xaf-skills
```

**Step 2** — install the plugin:
```shell
/plugin install xaf-devexpress@xaf-devexpress-skills
```

### Manual (global, all projects)

Clone the repo and copy skill folders to `~/.claude/skills/`:

```bash
git clone https://github.com/kashiash/xaf-skills
cp -r xaf-skills/xaf* ~/.claude/skills/
```

## Available Skills

| Skill | Topic |
|-------|-------|
| `xaf` | Master index — start here, links to all sub-skills |
| `xaf-xpo-models` | XPO persistent objects, base classes, attributes, associations |
| `xaf-ef-models` | EF Core entities, BaseObject, DbContext, migrations |
| `xaf-controllers` | Controllers, SimpleAction, PopupWindowShowAction, async patterns |
| `xaf-editors` | Built-in property editors, list editors, GridListEditor customization |
| `xaf-custom-editors` | Custom property/list editors for Blazor and WinForms |
| `xaf-nonpersistent` | NonPersistent objects, custom data sources, popup input forms |
| `xaf-security` | Security system, roles, permissions, OAuth2, JWT |
| `xaf-multi-tenant` | Multi-tenancy setup, ITenantProvider, per-tenant DB isolation |
| `xaf-web-api` | OData Web API, JWT auth, custom endpoints, $filter/$expand |
| `xaf-validation` | Validation rules, RuleCriteria, custom rules, programmatic validation |
| `xaf-reports` | XtraReports integration, predefined reports, export, designer |
| `xaf-dashboards` | Dashboard module, data sources, designer, permissions |
| `xaf-office` | File attachments, Spreadsheet, RichText, PDF viewer |
| `xaf-blazor-ui` | Blazor UI, InvokeAsync thread safety, custom Razor components |
| `xaf-winforms-ui` | WinForms UI, XtraGrid access, background workers, splash |
| `xaf-conditional-appearance` | [Appearance] attribute, hide/disable/color UI elements |
| `xaf-deployment` | IIS/Azure/Docker, ClickOnce, migrations, license, logging |

## Usage

Once installed, agents automatically discover these skills. To load a skill in Claude Code:

```
Use the xaf-controllers skill to help me create an action.
```

Or agents will auto-load relevant skills based on the task context.

## DevExpress MCP Server

For even deeper documentation access, add the DevExpress MCP server to `~/.claude/settings.json`:

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

## Structure

```
xaf-skills/
├── .claude-plugin/
│   └── marketplace.json        ← plugin marketplace catalog
├── xaf/SKILL.md                ← master index
├── xaf-xpo-models/SKILL.md
├── xaf-ef-models/SKILL.md
├── xaf-controllers/SKILL.md
└── ... (18 skill folders total)
```

## License

MIT — free to use, modify, and distribute.

## Contributing

PRs welcome. Each SKILL.md follows the format:

```yaml
---
name: xaf-topic-name
description: One-line description (max 1024 chars) for agent skill selection
license: MIT
compatibility: opencode, claude-code
---

# XAF: Topic Name
...
```

## Links

- [XAF Documentation](https://docs.devexpress.com/eXpressAppFramework/112670/expressapp-framework)
- [DevExpress MCP Server](https://community.devexpress.com/Blogs/news/archive/2025/10/16/transform-your-development-experience-with-the-devexpress-mcp-server.aspx)
- [DevExpress Support](https://supportcenter.devexpress.com)
- [XAF GitHub Examples](https://github.com/DevExpress-Examples)
