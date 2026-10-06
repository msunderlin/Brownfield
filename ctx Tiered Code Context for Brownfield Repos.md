# ctx: Tiered Code Context for Brownfield Repos

Oct 1, 2026 · @Mike

## Problem

AI agents spend most of their tokens orienting, not working, in our 1M+ line monolith. The code is organized by layer (controllers, services, listeners), so one business domain is scattered across directories, and grep returns hundreds of mentions for every name. Event-to-listener wiring lives in config, where text search rarely finds it.

Any fix must be tool-agnostic. Teams use different AI tools, and contractors bring their own. It should also be cheap to run and reusable for onboarding other legacy projects.

## Proposal

CI builds a small set of plain-text index files into `.context/`, and a thin CLI called `ctx` answers lookups from them. Agents pull only the slice they need, from the cheapest tier up.

| Tier | Contents | Budget |
| --- | --- | --- |
| L0 Project card | What the system does, stack, layer layout, domain list, conventions. Human-written. | 1-2k tokens, always loaded |
| L1 Domain index | Domain to file paths across all layers, built by clustering ctags symbol names | A few hundred tokens per domain |
| L2 Domain outline | Classes and public method signatures for one domain, no bodies | 2-5k tokens, capped |
| L3 Source | The code itself | Read on demand |

`ctx` is a 50-100 line script, not a service. Typical commands:

- `ctx def InvoiceService::finalize` returns file, line and signature in one line.
- `ctx domain invoice` returns every file in that domain, across layers.
- `ctx history InvoiceService::finalize` returns Jira tickets that changed it, mined from PR-prefixed merge commits.
- `ctx listeners InvoiceFinalized` returns the event's registered listeners.

Agents learn about it from a few lines in `AGENTS.md` or each tool's equivalent. Any tool with a shell can use it, so no per-tool integration is needed.

&#91;embedded content: ctx pipeline · generated and curated context, one CLI\]

Generated files and human-curated rules sit side by side in `.context/`; agents only ever see the short answers `ctx` returns.

## Token optimizations

The biggest saving is querying instead of preloading. These keep each answer small:

- **Compact formats, not JSON.** Group by file and class so each path appears once, and drop parameter names and defaults. That's roughly 3-4x smaller than per-symbol JSON.
- **Prune before ranking.** Leave out private methods, trivial accessors, `vendor/`, generated code and migrations. Tests get a separate index.
- **Rank, then truncate to budget.** Order by inbound mentions, git churn and public surface, and end with a marker such as `+37 more (ctx outline invoice --all)`.
- **Deterministic output.** Sorted, with no timestamps, so unchanged code yields byte-identical files. That keeps diffs clean and helps tools that cache prompts.
- **Curated beside generated.** A short human-written glossary and gotchas list carries the *why* per token better than anything generated. It lives in its own folder so regeneration never overwrites it.

Budgets use characters / 4 as the token estimate, so no vendor tokenizer is required.

## What the PHP ecosystem is doing

The Laravel and Symfony teams converged on the same shape this proposal uses: a small always-loaded instruction file, on-demand lookups, and deterministic tools instead of feeding the model raw code. Most of these packages need modern framework and PHP versions, so for legacy code we borrow the patterns, not the packages.

| Project | What it does | What we borrow |
| --- | --- | --- |
| [Laravel Boost](https://laravel.com/docs/boost) | Guidelines load upfront; skills load only when relevant to cut context bloat. Project rules are scoped by path glob, with an index agents check before editing. An `infer-conventions` skill sweeps existing code and asks a human to approve each convention with its evidence. | Glob-scoped rules index for our curated layer. Evidence-plus-approval as the onboarding step for each legacy project. |
| [Symfony Mate](https://symfony.com/blog/symfony-ai-spotlight-mate-an-agent-s-way-into-your-runtime) | Small CLI commands over the compiled container, profiler and logs, read from files Symfony already writes. In 0.13 it dropped its MCP server for a CLI plus `AGENTS.md` and skills, after a benchmark showed agents found the tools just as reliably. | Ship `ctx` as a CLI, not an MCP server. Read the framework's compiled container and route cache to cover the wiring ctags can't see. |
| [Laravel PAO](https://github.com/laravel/pao) | Detects an AI agent and replaces PHPUnit, PHPStan and Rector output with compact JSON. Claims up to 99.8% fewer tokens on test output. Requires PHP 8.3+. | Compact tool output. On older PHP, wrap our own test and PHPStan runs the same way. |
| [AWS AI-DLC](https://awslabs.github.io/aidlc-workflows/reference/04-stages/inception/) | Brownfield projects start with Reverse Engineering: an agent scans the whole codebase and writes 9 artifacts (architecture, components, APIs) to a per-repo store, with full or focused rescans. | Its artifacts become L0/L1 inputs. `ctx` gives its scan a cheap index, so a focused rescan covers one domain instead of reading 1M lines. |
| [Aider-style repo maps](https://github.com/dereira/goldfish) | Tree-sitter tags plus PageRank rank files and symbols, then a binary search fits the map to a token budget. | The ranking-and-budget approach for L2 outlines. |
| [Serena](https://github.com/oraios/serena) | Exposes a language server (Intelephense for PHP) as agent tools, including find-referencing-symbols and symbol-level edits. | The upgrade path when agents need real references, not just definitions. |

The gap for us is that Boost and Mate assume a current Laravel or Symfony app, and AI-DLC assumes an agent can scan the repo in one pass. A prebuilt index is what makes those patterns work on a 1M-line legacy codebase.

## Rollout and proving the benefit

Start with one repo and a measured pilot, then reuse the same tool and a per-project config for each additional legacy project.

1. **Build the minimum.** L0 card, `ctx def`, `ctx domain`, and a listener map read from config or the compiled container. CI regenerates them on merge.
2. **Infer and approve conventions.** Borrow Boost's approach: an agent proposes conventions with evidence, and an owner of the codebase approves each one into glob-scoped rules.
3. **Run a paired pilot.** Take 5-10 real, closed tickets and replay each with and without `ctx`, on the same model and tool. Record tokens, tool calls before the first correct file is opened, and whether the change passes review.
4. **Feed AI-DLC.** For teams already on AI-DLC, run Reverse Engineering with and without `ctx` in the instructions and compare token cost and artifact accuracy.
5. **Onboard the next project.** Write its config (languages, excludes, layer paths, domain aliases, event registration files, Jira key pattern), review the domain clusters, write L0, and add CI.

**Risks**

- Domain clustering by name is noisy. The human review in step 2 is required, not optional.
- Stale context is worse than none. Artifacts must regenerate in CI, and stale ones should fail loudly.
- Overbuilding before measuring. Ship the minimum and let pilot results pick the next feature.

**Open questions**

- Do any of the target apps have a compiled container or route cache we can read, or is all wiring hand-registered?
- Which ticket set is representative enough to convince other teams?
- Who owns the curated rules for each project after onboarding?

## Sources

- [Laravel Boost documentation](https://laravel.com/docs/boost)
- [Symfony AI Spotlight: Mate, an Agent's Way into Your Runtime](https://symfony.com/blog/symfony-ai-spotlight-mate-an-agent-s-way-into-your-runtime)
- [laravel/pao on GitHub](https://github.com/laravel/pao)
- [AI-DLC Inception phase stage reference](https://awslabs.github.io/aidlc-workflows/reference/04-stages/inception/)
- [goldfish: Go port of Aider's repo map](https://github.com/dereira/goldfish)
- [Serena on GitHub](https://github.com/oraios/serena)
