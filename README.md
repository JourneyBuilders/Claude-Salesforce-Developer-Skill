# Claude Salesforce Developer Skill

An experimental skill that teaches Claude how to work with Salesforce metadata without the constant back-and-forth caused by syntax errors, wrong API patterns, and deployment failures.

## The Problem

If you've used Claude (or any LLM) for Salesforce development, you know the loop: you ask it to generate a Flow, an Apex trigger, or a permission set — and then spend the next 5-10 iterations fixing XML namespace issues, wrong element ordering, Flow reference whitespace problems, and other metadata quirks that the model doesn't know about.

## The Experiment

This skill is a structured set of reference files that gets loaded into Claude's context window, giving it deep knowledge of:

- **SF CLI** commands and workflows
- **Apex** patterns, trigger frameworks, governor limits
- **Lightning Web Components** with correct syntax and lifecycle
- **SOQL/SOSL** query patterns and optimization
- **Flows** — including the XML structure that catches everyone off guard
- **Metadata validation** — common XML errors, deployment failures, and how to fix them
- **Debugging** — debug logs, Replay Debugger, Flow fault paths, LWC DevTools, Nebula Logger
- **Architecture** — Well-Architected Framework, decision guides, integration patterns
- **Security** — permission sets, field-level security, sharing rules
- **DevOps** — CI/CD, Git workflows, deployment strategies

The goal: **fewer iterations, fewer syntax errors, more working metadata on the first try.**

## How It Works

The skill follows the [Agent Skills open standard](https://agentskills.io) — a `SKILL.md` entrypoint with YAML frontmatter and 12 reference files in a `references/` folder. When loaded into Claude, it gives the model specific, actionable knowledge about Salesforce metadata patterns.

It doesn't try to replace Salesforce documentation. It focuses on the things that LLMs consistently get wrong: XML structure, element ordering, reference formats, deployment gotchas, and the undocumented quirks that only show up when you actually try to deploy.

## Installation

### Prerequisites

- A Claude **Pro**, **Max**, **Team**, or **Enterprise** subscription (Skills are not available on the free tier)
- **Code execution and file creation** enabled (Settings → Capabilities)

### Claude.ai (Web)

1. Download [`salesforce-developer.skill`](https://github.com/JourneyBuilders/Claude-Salesforce-Developer-Skill/raw/main/salesforce-developer.skill) from the repo
2. Go to **Customize** → **Skills**
3. Upload the `.skill` file
4. Make sure the skill toggle is **on**

The `.skill` file is a zip archive containing SKILL.md and all 12 reference files — no additional uploads needed.

Claude will automatically load the skill when it detects a Salesforce-related task. You can also say _"use the salesforce-developer skill"_ to invoke it explicitly.

> **Team / Enterprise:** Organization owners can provision the skill org-wide under Admin settings → Skills. It will then appear for all members automatically.

### Claude Desktop App

The Claude desktop app (macOS / Windows) uses the same skill system as claude.ai:

1. Download [`salesforce-developer.skill`](https://github.com/JourneyBuilders/Claude-Salesforce-Developer-Skill/raw/main/salesforce-developer.skill) from the repo
2. Open the Claude desktop app
3. Go to **Customize** → **Skills**
4. Upload the `.skill` file
5. Make sure the skill toggle is **on**

That's it — the skill works the same way across web and desktop.

### Claude Code (Terminal)

You can install the skill at two levels:

**Personal (available in all your projects):**

```bash
# Clone and copy to your personal skills directory
git clone https://github.com/JourneyBuilders/Claude-Salesforce-Developer-Skill.git
mkdir -p ~/.claude/skills/salesforce-developer
cp -r Claude-Salesforce-Developer-Skill/SKILL.md ~/.claude/skills/salesforce-developer/
cp -r Claude-Salesforce-Developer-Skill/references ~/.claude/skills/salesforce-developer/
```

**Project-level (available only in a specific project):**

```bash
# From your Salesforce project root
mkdir -p .claude/skills/salesforce-developer
cp -r /path/to/Claude-Salesforce-Developer-Skill/SKILL.md .claude/skills/salesforce-developer/
cp -r /path/to/Claude-Salesforce-Developer-Skill/references .claude/skills/salesforce-developer/
```

Your directory structure should look like this:

```
~/.claude/skills/                        # personal
  └── salesforce-developer/
      ├── SKILL.md
      └── references/
          ├── apex-patterns.md
          ├── architecture.md
          ├── data-queries.md
          ├── debugging.md
          ├── deployment-devops.md
          ├── flows-automation.md
          ├── integrations.md
          ├── lwc-guide.md
          ├── metadata-validation.md
          ├── security-permissions.md
          ├── sf-cli.md
          └── sources.md
```

Claude Code discovers skills automatically — no restart needed. You can verify it's loaded by asking Claude _"what skills do you have available?"_ or by checking if it references the skill in its chain of thought.

> **Tip:** If you install at the project level, commit `.claude/skills/` to your repo so the whole team gets it.

### Updating

**Claude.ai / Desktop:** Upload the new `.skill` file in Customize → Skills. It replaces the previous version.

**Claude Code:** Pull the latest from the repo and copy the files again:

```bash
cd Claude-Salesforce-Developer-Skill
git pull
cp -r SKILL.md ~/.claude/skills/salesforce-developer/
cp -r references ~/.claude/skills/salesforce-developer/
```

## Crowdsourced Error Logging

When this skill is installed, it encourages Claude to document deployment errors and their solutions as they happen. The idea is simple: every time you hit a metadata error and work through the fix, that's knowledge that should flow back into the skill so the next developer doesn't hit the same wall.

**How it works in practice:**

When you encounter a deployment error while using this skill, ask Claude to format it as a contribution:

> _"Log this deployment error and solution to the skill repo"_

Claude will generate a structured error report you can submit as a GitHub issue or PR:

```markdown
### Error: [error message]
**Metadata type:** Flow / Apex / LWC / PermissionSet / etc.
**Context:** What you were trying to do
**Root cause:** Why it failed
**Fix:** The actual solution
**Detection:** How to catch this before deploying
```

You can then submit it to [the repo](https://github.com/JourneyBuilders/Claude-Salesforce-Developer-Skill/issues/new) — we'll review and fold it into the appropriate reference file.

This is the core loop of the experiment: **deploy → fail → fix → document → share → prevent**. The more errors we collect and codify, the fewer iterations everyone needs.

## Does It Actually Help?

That's what we're trying to find out. Early results are promising — the skill catches a lot of the common metadata mistakes before they happen. But the real test is whether it holds up across the wide variety of Salesforce implementations that exist in the wild.

That's where you come in.

## Contributing

This skill gets better with every real-world deployment failure, every undocumented quirk, and every pattern that a developer has figured out the hard way. If you work with Salesforce metadata and have run into issues when using Claude (or other LLMs), we want to hear from you.

### Ways to contribute

**Report a problem** — If Claude generated bad metadata despite using this skill, open an issue with:

- What you asked for
- What Claude generated
- What the actual fix was
- The error message from the deployment

**Add a pattern** — If you know of a metadata quirk, deployment gotcha, or syntax requirement that isn't covered, submit a PR adding it to the relevant reference file.

**Share your results** — Even just telling us "this worked" or "this didn't help with X" is valuable. Open a discussion or drop a comment on an existing issue.

**Improve existing content** — If a reference file has outdated information, unclear explanations, or missing edge cases, PRs are welcome.

### Before contributing

- Check existing issues and reference files to avoid duplicates
- Keep entries concise and actionable — this is a reference, not a tutorial
- Include the actual error message when documenting a deployment failure
- Test your patterns against a real org if possible

> **⚠️ Security reminder:** Never include API tokens, passwords, session IDs, org credentials, OAuth secrets, or real customer data in issues or PRs. Sanitize all error output before posting. GitHub issues are public.

## Current State

This is an active experiment. The skill targets **API version 66.0 (Spring '26)** and covers the most common metadata types and deployment scenarios. It's nowhere near complete — Salesforce has hundreds of metadata types and thousands of ways things can go wrong.

The reference files total about 60 KB of compressed, actionable knowledge. That's small enough to fit in context alongside your actual work, but large enough to cover the patterns that cause 90% of the iteration loops.

## Built By

[JourneyBuilders](https://github.com/JourneyBuilders)

## License

MIT — use it, fork it, improve it, share it.
