# Contributing to XAF DevExpress Agent Skill

Thank you for contributing! This project follows the [Agent Skills](https://skills.sh) open format with specific structural requirements. Please read this guide before submitting a pull request.

## What Are Agent Skills?

Agent Skills are structured prompt resources containing a `SKILL.md` behavior file that provides procedural knowledge to AI coding assistants. Each skill is a self-contained folder with a `SKILL.md` file using YAML frontmatter.

## Using skill-creator for Contributions (Recommended)

We strongly recommend using AI assistance when creating or updating skills:

```bash
npx skills add vercel-labs/skills --skill skill-creator
```

Then in your AI agent:
```
Use the skill-creator skill to help me add a new section about XAF [topic] to the xaf-controllers skill.
```

This ensures:
- Correct SKILL.md frontmatter format
- Consistent style and tone with existing skills
- Proper source link structure
- Token-efficient, agent-readable content

## SKILL.md Format Requirements

Each skill file must follow this structure:

```yaml
---
name: xaf-topic-name          # must match folder name, lowercase-alphanumeric-hyphens
description: |                 # shown to agent for skill selection — max 1024 chars
  One-line summary of what this skill covers and when to use it.
license: MIT
compatibility: opencode, claude-code
metadata:
  domain: xaf
  topic: topic-name
  versions: v24.2, v25.1
---

# XAF: Topic Name

## Section Heading

Short description, then code example.

\`\`\`csharp
// Minimal, complete, copy-pasteable example
\`\`\`

---

## Source Links

- Topic name: https://docs.devexpress.com/...
```

### Key rules

- Folder name must match the `name` field in frontmatter
- `description` is used by AI agents to decide whether to load the skill — make it specific and keyword-rich
- All code examples must be compilable C# (no pseudocode)
- Every section should end with a source link to official DevExpress docs

## Acceptable Contributions

- Corrected or updated XAF code patterns
- New sections for features not yet covered
- Updated examples for new XAF versions (v25.1, v25.2, etc.)
- Improved explanations of common pitfalls
- Additional source links
- New skills for topics not yet covered

## Quality Standards

- **Focus**: Each skill covers one topic area only — no cross-cutting content
- **Code-first**: Lead with working C# examples, keep prose minimal
- **Accuracy**: Test patterns against actual XAF projects when possible
- **Versions**: Note if a feature is version-specific (e.g., "v24.2+")
- **Token efficiency**: Avoid redundancy — agents pay per token
- **Source links**: Every non-trivial pattern should link to official docs

## Adding a New Skill

1. Create a new folder: `xaf-your-topic/`
2. Add `SKILL.md` with correct frontmatter (name must match folder)
3. Add the skill path to `.claude-plugin/marketplace.json` if using explicit skills list
4. Reference the new skill in `xaf/SKILL.md` index table
5. Update the skills table in `README.md`

## Submission Steps

1. Fork the repository
2. Create a branch: `git checkout -b add-xaf-your-topic`
3. Make focused, scoped changes
4. Verify `SKILL.md` frontmatter is valid and name matches folder
5. Submit a PR with a short description of what was added or changed

## Helpful Links

- [Agent Skills format docs](https://skills.sh/docs)
- [XAF Documentation](https://docs.devexpress.com/eXpressAppFramework/112670/expressapp-framework)
- [XAF API Reference](https://docs.devexpress.com/eXpressAppFramework/DevExpress.ExpressApp)
- [DevExpress GitHub Examples](https://github.com/DevExpress-Examples)
- [skill-creator skill](https://skills.sh/vercel-labs/skills/skill-creator)

## Community Standards

Engage respectfully, assume good faith, and focus on improving the quality of XAF guidance for the developer community.
