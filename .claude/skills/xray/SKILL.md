---
name: xray
description: Behavioral code analysis using git history. Use when the user asks to "find hotspots", "analyze technical debt", "find change coupling", "show complexity trend", "code age analysis", "knowledge map", "developer fragmentation", "rising hotspots", "knowledge loss", or wants to prioritize refactoring targets using version-control data. Based on techniques from "Software Design X-Rays" by Adam Tornhill.
allowed-tools: [Bash, Read, Glob, Grep]
argument-hint: <subcommand> [args] — hotspots | trend <file> | xray <file> | coupling | age | knowledge | fragmentation | rising | knowledge-loss <author> | summary
---

# X-Ray: Behavioral Code Analysis

Analyze any git repository to uncover technical debt, coordination bottlenecks, and architectural issues that are invisible in the code itself. All analyses mine version-control history and work on any programming language with zero external dependencies beyond git.

When available, the skill uses **cloc** (accurate language-aware line counts) and **Code Maat** (Tornhill's open-source analysis engine) for enhanced results. When these tools are not installed, it falls back to git + shell commands.

## Usage

```
/xray hotspots                    Find files with highest interest rate (change freq x size)
/xray trend <file>                Show complexity trend for a file over time
/xray xray <file>                 Function-level hotspot analysis inside a file
/xray coupling                    Detect files that change together (change coupling)
/xray age                         Map code age; find mixed-age packages to refactor
/xray knowledge                   Knowledge map: main developer per file/component
/xray fragmentation               Developer diffusion analysis (fractal values)
/xray rising                      Early warning: files climbing the hotspot ranking
/xray knowledge-loss <author>     Simulate impact of a developer leaving
/xray summary                     Combined report: hotspots + coupling + fragmentation
/xray visualize hotspots           Generate interactive D3.js enclosure diagram (opens in browser)
/xray visualize coupling           Generate interactive D3.js edge bundle diagram (opens in browser)
```

## Routing

Parse `$ARGUMENTS` to route to the correct subcommand:

**User invocation:** $ARGUMENTS

1. Split arguments on whitespace. First token = subcommand. Remaining tokens = subcommand args.
2. Read `reference.md` in this skill directory (`${CLAUDE_SKILL_DIR}/reference.md`).
3. Find the section matching the subcommand and follow the procedure described there exactly.
4. If no subcommand is given or it is unrecognized, display the usage table above.

## Global Defaults

Apply these defaults to every analysis unless the user overrides them:

**Time windows:**
- Technical analyses (hotspots, trend, xray, coupling, age, rising): last **6 months**
- Social analyses (knowledge, fragmentation, knowledge-loss): last **3 months**
- Compute the `--after=YYYY-MM-DD` date from today's date

**Override flags** (can appear anywhere in arguments):
- `--since=YYYY-MM-DD` — set explicit start date
- `--months=N` — set analysis window to N months from today
- `--top=N` — limit results to top N entries
- `--exclude=<glob>` — add extra exclusion pattern
- `--path=<dir>` — scope analysis to a subdirectory

**File exclusions** — always filter these from results:
`*.min.js`, `*.min.css`, `package-lock.json`, `yarn.lock`, `Cargo.lock`, `poetry.lock`, `*.lock`, `go.sum`, `*.generated.*`, `*.pb.go`, `*.g.dart`, `*_generated.*`, `vendor/*`, `node_modules/*`, `dist/*`, `build/*`, `*.svg`, `*.png`, `*.jpg`, `*.gif`, `*.ico`, `*.woff*`, `*.ttf`, `*.eot`

Use `--no-merges` on all git log commands to avoid counting merge commits.

**Metadata header** — start every analysis output with:
```
Repository: <repo name from basename of git root>
Analysis period: YYYY-MM-DD to YYYY-MM-DD
Total commits in period: N (from git rev-list --count --after=<date> --no-merges HEAD)
```

## Ethics Warning

For the `knowledge`, `fragmentation`, and `knowledge-loss` subcommands, always include this notice at the top of the output:

> **Note:** These analyses reveal how developers interact with code. They are for knowledge management, communication, and risk assessment only. They MUST NOT be used for performance evaluation. Individual contribution patterns reflect situational factors — code complexity, team structure, domain difficulty, pair programming — that are invisible in the data. Using this data for evaluation destroys team dynamics and biases the data source itself.

## Output Principles

1. Use markdown tables for all ranked results
2. End every analysis with an **Actionable Recommendations** section containing specific, concrete suggestions
3. When a file appears in multiple analyses (e.g., hotspot AND fragmented), explicitly call out this intersection as highest priority
4. Use text risk labels: `CRITICAL`, `HIGH`, `MEDIUM`, `LOW` — no emoji
5. When presenting hotspot scores, normalize to a 0-1 scale for comparability
6. Always note limitations: renamed files may appear as separate entries; squash merges hide coupling data; pair programming biases author counts

## Tool Detection

At the start of any analysis, check for enhanced tooling:
```
which cloc && echo "cloc available" || echo "cloc not found — using wc -l (install: brew install cloc)"
which java && test -f code-maat-standalone.jar && echo "Code Maat available" || echo "Code Maat not found — using git+shell (optional: https://github.com/adamtornhill/code-maat)"
```
- If `cloc` is found: use it instead of `wc -l` for all LOC measurements
- If Code Maat JAR is found: it can be used for coupling and author analyses (see reference.md)
- Always report which tools were used at the bottom of the output

## Visualization

The `visualize` subcommand generates interactive HTML files using embedded D3.js and opens them in the default browser. No npm/node required — the HTML is self-contained with D3 loaded from CDN.

Available visualizations:
- `visualize hotspots` — **Enclosure diagram**: nested circles following folder hierarchy. Circle size = LOC, color intensity = change frequency (deeper red = more changes). Click to zoom into subfolders.
- `visualize coupling` — **Hierarchical edge bundle**: files as nodes arranged by directory, change coupling shown as curved links between them. Hover to highlight a file's dependencies.

See the `visualize` section in `reference.md` for the HTML generation procedure.
