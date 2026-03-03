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

The skill is a `.skill` file (a zip archive) containing a `SKILL.md` entrypoint and 12 reference files in a `references/` folder. When loaded into Claude — either via Claude Code's skills directory or as context in Claude.ai — it gives the model specific, actionable knowledge about Salesforce metadata patterns.

It doesn't try to replace Salesforce documentation. It focuses on the things that LLMs consistently get wrong: XML structure, element ordering, reference formats, deployment gotchas, and the undocumented quirks that only show up when you actually try to deploy.

## Installation

### Claude Code

Drop the extracted folder into your project's skills directory:

```
your-project/
├── .claude/
│   └── skills/
│       └── salesforce-developer/
│           ├── SKILL.md
│           └── references/
│               ├── apex-patterns.md
│               ├── architecture.md
│               ├── data-queries.md
│               ├── debugging.md
│               ├── deployment-devops.md
│               ├── flows-automation.md
│               ├── integrations.md
│               ├── lwc-guide.md
│               ├── metadata-validation.md
│               ├── security-permissions.md
│               ├── sf-cli.md
│               └── sources.md
```

### Claude.ai

Add the contents of `SKILL.md` to your user preferences or project instructions, with the relevant reference files attached as context.

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
