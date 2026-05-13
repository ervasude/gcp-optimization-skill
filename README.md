# GCP Optimization Skill

A modular, agent-readable knowledge skill for **cost and performance optimization on Google Cloud Platform**. All content is derived from Google Cloud's official documentation and structured for use by both humans and LLM agents (Claude, Cursor, Copilot, etc.).

## What is this?

This is a [Skill](https://www.anthropic.com/news/skills) — a structured Markdown knowledge base that LLM agents can read on demand. The format follows the Anthropic Skills convention (`SKILL.md` + supporting reference files), but the content itself is plain Markdown and works with any agent that reads instruction files.

The skill helps answer practical questions like:

- Why is this BigQuery query slow or expensive?
- Should I partition or cluster this table — and on what column?
- Why is my Cloud Storage bill growing without obvious reason?
- What's the right storage class for this data?
- How do I read a BigQuery query plan to find the bottleneck?

## Coverage

### BigQuery (`references/bigquery.md`)

1. **Partition Pruning** — partition basics, when filters trigger pruning, cost impact
2. **Clustering** — how clustering differs from partitioning, column selection rules, type constraints
3. **`SELECT *` Avoidance** — columnar storage explanation, `SELECT * EXCEPT` patterns
4. **Materialized Views** — when to use, cost-benefit tradeoff, limitations and gotchas
5. **JOIN Optimization** — order, broadcast joins, pre-filtering, data type matching
6. **Query Plan & Slot Analysis** — reading execution details, identifying bottlenecks (skew, shuffle, wait time)
7. **Storage Optimization** — active vs long-term, partition/table expiration, time travel, billing models

### Cloud Storage (`references/cloud-storage.md`)

1. **Storage Classes** — Standard, Nearline, Coldline, Archive
2. **Lifecycle Management** — automatic transitions and deletion patterns
3. **Object Versioning** — the hidden cost trap and how to mitigate it
4. **Location** — region vs dual-region vs multi-region, egress considerations
5. **Quick Wins** — compression, object size, audit patterns

## How to use

### As a human

Open `SKILL.md` to see the index, then open the relevant `references/*.md` file. Each section follows the same structure: concept → why it matters → bad/good example → reference link to the official Google Cloud doc.

### With Claude (claude.ai / Claude Code)

Place the entire `gcp-optimization-skill/` folder under your skills directory. Claude reads `SKILL.md` automatically and pulls in the reference files when relevant.

### With Cursor

Copy the contents into `.cursor/rules/` as `.mdc` files, or reference them with `@` in chat.

### With GitHub Copilot

Reference `SKILL.md` from `.github/copilot-instructions.md`, or copy the content directly into that file.

### With other agents (Windsurf, Cline, Continue, etc.)

The Markdown is portable — drop it into whatever rules/instructions directory the tool uses.

## Design principles

- **Source-grounded.** Every section links back to the Google Cloud documentation page it summarizes. No fabricated facts.
- **Concise over exhaustive.** Each section answers a specific question in 3-5 sentences, then links to the source for depth. Agents waste fewer tokens; humans skim faster.
- **Modular.** Top-level `SKILL.md` stays short. Detailed content lives in topic files that can be loaded individually.
- **Tool-agnostic.** No agent-specific syntax, no hardcoded paths. Plain Markdown that any reader (LLM or human) can use.

## Status

Built as part of an internship project. BigQuery and Cloud Storage sections are complete. Compute and Data Pipelines sections are stubbed and will be filled in as the skill grows.

Contributions, corrections, and pull requests are welcome.

## License

Documentation content is summarized from Google Cloud's official docs (CC BY 4.0). This compilation is released under the MIT License — see `LICENSE` for details.