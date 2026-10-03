---
name: defprod-exhaustive-discussion
description: Conduct a structured, high-rigor design review by systematically exploring and resolving each branch of a design decision tree. Use when stress-testing or deeply refining a plan — and as the depth tier the change-design stage escalates to for large or ambiguous changes.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - AskUserQuestion
  - mcp__defprod__listProducts
  - mcp__defprod__getProduct
  - mcp__defprod__listAreas
  - mcp__defprod__getArea
  - mcp__defprod__listUserStories
  - mcp__defprod__getUserStory
  - mcp__defprod__getBriefForProduct
  - mcp__defprod__getArchitectureForProduct
---

> **Local extensions.** If a file named `SKILL.local.md` exists in this skill's
> directory, read it now and fold it into the steps below. It records this
> installation's local policies, additions, and overrides; where it conflicts
> with the instructions here, the local file takes precedence.

Drive the conversation like a senior architect performing a design review.

Systematically walk the design tree:
- Identify the most critical or highest-risk unresolved decision
- Explore that branch from high-level → detailed decisions
- Resolve dependencies between decisions before proceeding
- Do not jump between unrelated branches

For each step:
- Ask ONE high-leverage question at a time
- Explain why the question matters
- Provide your recommended answer with reasoning
- Challenge the user's assumptions with constructive, evidence-based counterpoints
- Clearly state trade-offs (pros/cons)

Maintain structure throughout:
- Keep track of decisions made
- Call out unresolved questions
- Periodically summarise the current state of the design

Available information sources:
- Documentation, particularly design documentation in the "docs" directory
- The product specification in the DefProd MCP server, if available
- The codebase

Use available sources proactively:
- Check them BEFORE asking questions if they may already contain answers
- Use them to validate or challenge assumptions

Aim for convergence:
- Continue until all major branches are resolved or explicitly deferred
- End with a clear summary of:
  - Final decisions
  - Key trade-offs
  - Remaining risks / unknowns

Finally:
  - Ask if the ideas discussed and conclusions chosen need to be stored in a permanent document
