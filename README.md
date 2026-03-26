# `/xray` — Behavioral Code Analysis for Claude Code

A [Claude Code](https://claude.ai/claude-code) skill that mines git history to uncover technical debt, coordination bottlenecks, and architectural issues invisible in the code itself. Based on techniques from [Software Design X-Rays](https://pragprog.com/titles/atevol/software-design-x-rays/) by Adam Tornhill.

Works on **any git repository**, regardless of programming language. Zero external dependencies beyond git.

## Install

Copy the skill into your Claude Code skills directory:

```bash
# Clone this repo
git clone https://github.com/iledzka/xray-skill.git

# Copy to your global Claude Code skills
mkdir -p ~/.claude/skills
cp -r xray-skill/.claude/skills/xray ~/.claude/skills/xray
```

Or for a single project only:

```bash
mkdir -p .claude/skills
cp -r xray-skill/.claude/skills/xray .claude/skills/xray
```

## Usage

In Claude Code, `cd` into any git repository and run:

```
/xray hotspots                     Find files with highest interest rate (change freq × size)
/xray trend <file>                 Show complexity trend for a file over time
/xray xray <file>                  Function-level hotspot analysis inside a file
/xray coupling                     Detect files that change together (change coupling)
/xray age                          Map code age; find mixed-age packages to refactor
/xray knowledge                    Knowledge map: main developer per file/component
/xray fragmentation                Developer diffusion analysis (fractal values)
/xray rising                       Early warning: files climbing the hotspot ranking
/xray knowledge-loss <author>      Simulate impact of a developer leaving
/xray summary                      Combined report: hotspots + coupling + fragmentation
/xray visualize hotspots           Interactive D3.js enclosure diagram (opens in browser)
/xray visualize coupling           Interactive D3.js edge bundle diagram (opens in browser)
```

## What each command does

### Start here

**`/xray hotspots`** — The foundation. Finds files that are both large and frequently changed. These are where refactoring pays off the most. Development activity follows a power-law distribution: a small fraction of files attract the majority of commits.

### Dig deeper into a hotspot

**`/xray trend <file>`** — Shows how a file's complexity has changed over time using indentation-based complexity (language-neutral). A rising trend means the code is getting harder to understand with each change.

**`/xray xray <file>`** — Function-level analysis inside a file. Narrows a 2000-line hotspot down to the specific 50-200 lines that need attention.

### Find hidden dependencies

**`/xray coupling`** — Detects files that change together in the same commits. Surprising cross-directory coupling reveals implicit dependencies, missing abstractions, and copy-paste code.

### Understand code evolution

**`/xray age`** — Maps how old each file is (time since last modification). Highlights directories with mixed-age code — refactoring candidates where stable code could be extracted into libraries.

### Social analysis

**`/xray knowledge`** — Shows the main developer behind each file. Identifies expertise domains, bus-factor risks, and communication paths.

**`/xray fragmentation`** — Measures how diffused development effort is across contributors. Files with many minor contributors and no clear owner have higher defect risk.

**`/xray knowledge-loss <author>`** — Simulates the impact of a developer leaving. Shows which files would lose their primary knowledge holder.

### Early warnings

**`/xray rising`** — Compares hotspot rankings across two time periods to find files that are climbing. These are future maintenance problems developing right now.

### Overview

**`/xray summary`** — Runs hotspots + coupling + fragmentation and cross-references the results into a single risk-prioritized report with actionable recommendations.

### Visualizations

**`/xray visualize hotspots`** — Generates an interactive enclosure diagram (nested circles). Circle size = lines of code, color = change frequency. Click to zoom into subdirectories. Opens in browser.

**`/xray visualize coupling`** — Generates an interactive hierarchical edge bundle. Files arranged by directory with curved links showing coupling. Red = surprising, blue = expected. Hover to highlight.

## Optional enhanced tools

The skill works with just `git`, but produces better results when these are available:

- **[cloc](https://github.com/AlDanial/cloc)** — Language-aware line counting (excludes comments and blanks). Install: `brew install cloc`
- **[Code Maat](https://github.com/adamtornhill/code-maat)** — Adam Tornhill's open-source analysis engine. Provides more robust coupling and author analysis.

## Options

Append these flags to any subcommand:

| Flag | Description |
|------|-------------|
| `--since=YYYY-MM-DD` | Set explicit start date |
| `--months=N` | Set analysis window to N months |
| `--top=N` | Limit results to top N entries |
| `--path=<dir>` | Scope analysis to a subdirectory |
| `--exclude=<glob>` | Add extra file exclusion pattern |

## How it works

All analyses mine your git history — the commits, authors, and file changes already recorded in your version control system. No code is sent anywhere. The key insight from Tornhill's work: your version-control data is a behavioral log that reveals how your team actually works with the code, which is information you cannot get from the code itself.

## Credits

Based on [Software Design X-Rays](https://pragprog.com/titles/atevol/software-design-x-rays/) by Adam Tornhill (Pragmatic Bookshelf, 2018). The book covers these techniques in much greater depth and is highly recommended.

Adam Tornhill also created [CodeScene](https://codescene.com/) — the full-featured commercial implementation of these techniques with a web UI, CI/CD integration, historical tracking, team dashboards, and much more. If you need behavioral code analysis at scale or across an organization, CodeScene is the real deal. This skill is a lightweight CLI companion for quick, local exploration inspired by the book's teachings.

## License

MIT
