# x-for-you-algorithm — an Agent Skill

[![skills.sh](https://skills.sh/b/kkkhushman/x-algo-skill)](https://skills.sh/kkkhushman/x-algo-skill)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

A portable, deep-dive **knowledge skill** that equips any AI coding agent (Claude Code, Cursor, Codex, Copilot, Gemini, OpenCode, and 50+ others) to reason correctly about **X's open-sourced For You feed algorithm** (`xai-org/x-algorithm`, released 2026-05-15).

## What this skill is

This is a **reference skill**, not a workflow. It encodes everything an agent needs to know to:

- Explain how X ranks posts in the For You feed.
- Design or audit a feed recommendation system using X's architectural choices.
- Build on top of the For You codebase (porting, extending, integrating).
- Compare the 2026 X algorithm against the 2023 Twitter algorithm.
- Answer specific questions about the Phoenix model, Thunder, home-mixer, candidate-pipeline, Grox, or ads blending.

Every claim cites the upstream `xai-org/x-algorithm` repo by `file:line`, so an agent can verify before acting.

## Install

Using the [skills.sh](https://skills.sh) CLI:

```bash
# Install to the current project for any detected agents
npx skills add kkkhushman/x-algo-skill

# Install globally for Claude Code only
npx skills add kkkhushman/x-algo-skill -g -a claude-code

# List the skill's contents before installing
npx skills add kkkhushman/x-algo-skill --list
```

For Claude Code specifically you can also clone directly:

```bash
git clone https://github.com/kkkhushman/x-algo-skill ~/.claude/skills/x-for-you-algorithm
```

## What's inside

```
.
├── SKILL.md                                 # entrypoint — read this first
├── references/
│   ├── 01-architecture.md                   # system view, gRPC surfaces, data flow
│   ├── 02-phoenix-model.md                  # transformer + two-tower + masking + hash embeddings
│   ├── 03-request-lifecycle.md              # end-to-end For You request trace
│   ├── 04-pipeline-framework.md             # the candidate-pipeline trait system
│   ├── 05-ranking-and-blending.md           # 22 weights, author diversity, safe-gap ads
│   ├── 06-content-understanding.md          # Thunder, Grox, all the filters
│   ├── 07-common-pitfalls.md                # 22 ways agents reason wrongly about this
│   └── 08-glossary.md                       # LAP7, FOU, Gizmoduck, Strato, VQV, UPA, etc.
├── LICENSE                                  # Apache-2.0
└── README.md                                # this file
```

## When this skill triggers

Common scenarios where an agent should load this skill:

- "Explain X's For You feed algorithm"
- "How does Phoenix rank tweets?"
- "Build a recsys like Twitter / X"
- "What's the difference between the 2023 Twitter algorithm and the 2026 X release?"
- "Why does my Phoenix port not match the upstream behavior?"
- Questions naming any of: `Phoenix`, `candidate isolation`, `two-tower retrieval`, `home-mixer`, `Thunder`, `Grox`, `candidate-pipeline`, `SafeGapAdsBlender`, `xai-org/x-algorithm`, `For You feed`
- Errors / confusion around: `weighted_score vs score`, `19 actions vs 15`, `why are filters sequential`, `why no FAISS in retrieval`

## Compatibility

Works on any agent in the [skills.sh supported list](https://github.com/vercel-labs/skills#supported-agents) — Claude Code, Cursor, Codex, OpenCode, Gemini CLI, GitHub Copilot, Continue, Cline, Codebuddy, Crush, Droid, Goose, Junie, Kiro, Qwen, Roo Code, Trae, Windsurf, Zencoder, and ~30 more.

Skill format follows the [Agent Skills specification](https://agentskills.io): a `SKILL.md` with YAML frontmatter (`name`, `description`) at the root, plus supporting markdown loaded on demand.

## License

This skill is licensed Apache-2.0, matching `xai-org/x-algorithm`. The upstream source code at `https://github.com/xai-org/x-algorithm` is the authoritative reference — this skill is a curated, citation-grounded map of it, not a replacement.

## Maintenance

- Built against the `xai-org/x-algorithm` repo as of the **May 15, 2026** release.
- Citations use `file:line` references that may drift if the upstream repo is updated. If you spot a citation that no longer matches, open an issue.

---

Built with care by [@kkkhushman](https://github.com/kkkhushman).
