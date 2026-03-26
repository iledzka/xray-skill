# X-Ray Reference: Algorithms and Git Pipelines

This document contains the exact procedures for each `/xray` subcommand. Follow the steps precisely. All git commands assume you are in the root of a git repository.

Notation: `<date>` refers to the computed `--after` date from SKILL.md global defaults.

---

## hotspots

### Purpose
Find files with the highest "interest rate" — code that is both large and frequently changed. These are the files where improvements yield the most return on investment. Development activity follows a power-law distribution: a small fraction of files attract the majority of commits.

### Procedure

**Step 1: Get change frequencies**
```bash
git log --format=format: --name-only --no-merges --after="<date>" | egrep -v '^$' | sort | uniq -c | sort -rn | head -100
```

**Step 2: Filter out non-code artifacts**
Remove files matching the exclusion patterns from SKILL.md. Also remove files that no longer exist on disk (deleted files appear in git history).

**Step 3: Get lines of code for top files**
For each of the top 50 remaining files:
```bash
wc -l "<file>"
```
If `cloc` is available (check with `which cloc`), prefer it for more accurate code-only line counts.

**Step 4: Calculate hotspot score**
For each file: `score = change_frequency × lines_of_code`. Normalize scores to a 0.0–1.0 range by dividing by the maximum score.

**Step 5: Present results**
```
## Hotspot Analysis

| Rank | File | Changes | LOC | Score |
|------|------|---------|-----|-------|
| 1    | path/to/file.ext | 142 | 3200 | 1.00 |
| ...  | ... | ... | ... | ... |
```

### Interpretation Guide
- Top hotspots are candidates for refactoring — start with complexity trend (`/xray trend`) and X-Ray (`/xray xray`) on the #1 hotspot
- Test files appearing as hotspots are common and indicate test code receiving less design attention than application code — treat them with equal care
- Hotspots tend to persist for years unless actively refactored
- A file being a hotspot is not inherently bad — it means improvement there has the highest payoff

### Actionable Recommendations Template
- For hotspots > 500 LOC: suggest running `/xray xray <file>` to identify function-level targets
- For hotspots > 1000 LOC: suggest splinter refactoring (break the file along its responsibilities while maintaining the original API as a facade)
- For test file hotspots: suggest looking for duplicated test data and missing test abstractions

---

## trend

### Purpose
Show how a file's complexity has changed over time using indentation-based complexity — a language-neutral metric that correlates well with traditional complexity measures like cyclomatic complexity. A rising trend means the code is getting harder to understand with each change.

### Procedure

**Step 1: Get revision history**
```bash
git log --format="%H %ad" --date=short --no-merges --after="<date>" -- "<file>"
```
This gives commit hashes and dates. If more than 25 revisions, sample evenly to get ~20 data points.

**Step 2: Calculate complexity for each revision**
For each sampled commit hash:
```bash
git show <hash>:"<file>" 2>/dev/null
```
Then calculate indentation-based complexity on the output:
- For each line, count leading whitespace characters (expand tabs to 4 spaces)
- **Total complexity** = sum of all indentation levels across all non-blank lines
- **Lines** = total non-blank lines
- **Average complexity** = total complexity / lines
- **Max depth** = maximum indentation level found in any single line

**Step 3: Detect trend direction**
Compare first-third average complexity to last-third average complexity:
- If last third is >15% higher → `INCREASING` (warning)
- If last third is >15% lower → `DECREASING` (improvement)
- Otherwise → `STABLE`

**Step 4: Present results**
```
## Complexity Trend: <file>

| Date | Lines | Total Complexity | Avg/Line | Max Depth |
|------|-------|-----------------|----------|-----------|
| 2024-01-15 | 340 | 1250 | 3.7 | 8 |
| ... | ... | ... | ... | ... |

Trend: INCREASING ⚠
The file's complexity has grown by X% over the analysis period while lines grew by Y%.
When complexity grows faster than size, it indicates patches with nested conditionals
rather than well-structured additions.
```

### Interpretation Guide
- Complexity growing faster than line count = nested conditionals being shoehorned in (bad)
- Complexity growing proportionally to line count = new functionality added in a structured way (neutral)
- Sudden spikes followed by stability = likely a mid-project formatting or indentation style change (false positive — note this)
- A decreasing trend after a spike = successful refactoring (celebrate this)

---

## xray

### Purpose
Function-level hotspot analysis inside a single file. Identifies which functions within a hotspot are responsible for most of the change activity. This narrows a 2000-line hotspot down to the specific 50-200 lines that need attention.

### Procedure

**Step 1: Detect the file's language**
Infer from file extension. Needed to choose function-parsing strategy.

**Step 2: Extract function boundaries**
Parse the current version of the file to find function/method definitions. Use `grep -n` with language-appropriate patterns:

- **Python:** `grep -n "^\s*def \|^\s*class " "<file>"`
- **JavaScript/TypeScript:** `grep -n "function \|=> {$\|^\s*\(export \)\?\(default \)\?\(async \)\?function\|^\s*[a-zA-Z_][a-zA-Z0-9_]*\s*(.*)\s*{" "<file>"`
- **Java/C#/Kotlin:** `grep -n "^\s*\(public\|private\|protected\|static\|override\|fun \|void \|int \|String \|bool \|async \)" "<file>"`
- **Go:** `grep -n "^func " "<file>"`
- **Ruby:** `grep -n "^\s*def " "<file>"`
- **Rust:** `grep -n "^\s*\(pub \)\?fn " "<file>"`
- **C/C++:** `grep -n "^[a-zA-Z_].*(.*).*{" "<file>"`

For each match, record the function name and start line.

**Step 3: Use git log -L for change counts (preferred method)**
For each detected function, count how many commits touched it:
```bash
git log --oneline --no-merges --after="<date>" -L:"<function_name>":"<file>" 2>/dev/null | grep -c "^[0-9a-f]"
```
Note: `git log -L` uses its own function detection heuristic. If it fails for a specific function, fall back to the diff-mapping approach.

**Step 3 (fallback): Map diff hunks to functions**
```bash
git log --format="COMMIT:%H" -p --no-merges --after="<date>" -- "<file>"
```
Parse `@@` hunk headers to extract modified line ranges. Map each hunk to the function whose line range contains the modified lines. Count commits per function.

**Step 4: Get function lengths**
For each function, calculate its length in lines (from its start line to the start of the next function, or end of file).

**Step 5: Calculate function hotspot score**
`score = change_count × function_length`. Sort descending.

**Step 6: Present results**
```
## X-Ray: <file>
Total file changes in period: N

| Rank | Function | Changes | Length (lines) | Score |
|------|----------|---------|----------------|-------|
| 1    | createInvoker | 45 | 197 | 1.00 |
| ... | ... | ... | ... | ... |
```

### Interpretation Guide
- Top function is the primary refactoring target — run `/xray trend` scoped to understand if its complexity is growing
- Functions > 50 lines with high change frequency: candidates for Extract Method refactoring to create brain-friendly chunks
- Multiple overloaded methods with the same name group together as one logical unit
- Look for control coupling (Boolean parameters), excess mocking in tests, and deeply nested conditionals

### Actionable Recommendations Template
- For functions > 100 lines: suggest splitting into smaller chunks using Extract Method
- For functions with many Boolean parameters: suggest replacing control coupling with polymorphism or strategy pattern
- Suggest proximity refactoring: move cochanging functions closer together in the file

---

## coupling

### Purpose
Detect change coupling — files that change together in the same commits. Surprising cross-directory coupling reveals implicit dependencies, missing abstractions, and copy-paste code. Expected coupling (test + source) confirms healthy practices.

### Procedure

**Step 1: Extract commit-file groups**
```bash
git log --format="COMMIT:%H" --name-only --no-merges --after="<date>"
```
Parse this output into groups: each `COMMIT:` line starts a new group, followed by the files changed in that commit. Skip commits with only 1 file (no coupling possible). Skip commits with >30 files (likely bulk operations or reformatting).

**Step 2: Generate all file pairs per commit**
For each commit group with 2-30 files, generate all unique pairs (A, B) where A < B alphabetically.

**Step 3: Count co-occurrences**
For each pair, count how many commits they appear together in. Also count total commits for each individual file (from the hotspot frequency data).

**Step 4: Calculate coupling degree**
```
degree = co_occurrences / min(total_changes_A, total_changes_B) × 100
```
Using `min` rather than `max` gives a more conservative measure — the coupling percentage relative to the file that changed less often.

**Step 5: Filter**
Keep only pairs where:
- `co_occurrences >= 10`
- `degree >= 30%`

**Step 6: Classify and sort**
- **Surprising:** Files are in different parent directories AND are not an obvious test/source pair (don't share a base name)
- **Expected:** Files share a base name (e.g., `foo.ts` and `foo.test.ts`), or are in the same directory

Sort surprising coupling first, then by degree descending.

**Step 7: Present results**
```
## Change Coupling Analysis

Commits analyzed: N | Pairs above threshold: N

### Surprising Coupling (cross-boundary)
| File A | File B | Co-changes | Degree | Classification |
|--------|--------|------------|--------|----------------|

### Expected Coupling
| File A | File B | Co-changes | Degree |
|--------|--------|------------|--------|
```

### Interpretation Guide
- Surprising coupling between files in different directories = likely a missing abstraction, copy-paste code, or a leaky abstraction
- Very high coupling (>80%) between unrelated files = investigate with `/xray xray` on both files to find the cochanging functions
- Expected test/source coupling below 50% = tests may not be kept up to date with the code they test
- A cluster of 3+ files all coupled to each other = strong candidate for extracting into a shared module

---

## age

### Purpose
Map the age of code across the codebase. Old stable code is low-risk — it has been tested by time. Recent code is fresh in developers' minds. Mid-age code is dangerous: we've forgotten details but still need to modify it. Packages with mixed-age code are refactoring candidates.

### Procedure

**Step 1: Get all tracked files**
```bash
git ls-files
```
Filter out excluded patterns.

**Step 2: Get last modification date per file (batch approach)**
Rather than running git log per file (slow), use:
```bash
git log --format="%ad %H" --date=short --name-only --no-merges --diff-filter=AMRC | head -5000
```
Parse this to build a map of file → most recent date. For files not appearing (unchanged in the log window), use:
```bash
git log -1 --format="%ad" --date=short -- "<file>"
```

**Step 3: Calculate age in days**
From today's date minus last modification date.

**Step 4: Categorize**
- **Recent:** < 30 days
- **Mid-age:** 30–180 days
- **Old:** > 180 days

**Step 5: Aggregate by directory**
For each directory, count files in each age category. Calculate a **mix score**: higher when a directory contains both recent and old files. `mix_score = 1 - (max_category_count / total_files)`. Score of 0 = all files same age. Score approaching 1 = evenly mixed.

**Step 6: Present results**
```
## Code Age Map

### Age Distribution
| Category | Files | Percentage |
|----------|-------|------------|
| Recent (< 1 month) | N | X% |
| Mid-age (1-6 months) | N | X% |
| Old (> 6 months) | N | X% |

### Mixed-Age Directories (refactoring candidates)
| Directory | Recent | Mid | Old | Total | Mix Score |
|-----------|--------|-----|-----|-------|-----------|
```

### Interpretation Guide
- Directories with high mix scores contain code changing at different rates — consider separating stable code into a library
- Old code that is also a hotspot = code we've been afraid to touch but keep patching around
- Mid-age code by a developer who has since left = "ignorant surgery" risk — modifications without understanding original design intent
- A directory where all code is old and stable = candidate for extraction into a standalone library or package

---

## knowledge

### Purpose
Build a knowledge map showing the main developer behind each file or module. Identifies expertise domains, bus-factor risks, and communication paths. Main developer = the person who has written the most code in the analysis period.

### Procedure

**Step 1: Get author contributions across all files**
```bash
git log --format="%aN" --numstat --no-merges --after="<date>"
```
Parse this output: each author name line is followed by numstat lines (`additions\tdeletions\tfilename`). Sum additions per author per file.

Alternative (simpler, commit-count based):
For the top 50 hotspot files, run per file:
```bash
git shortlog -sn --no-merges --after="<date>" -- "<file>"
```
This gives commit counts per author for that file.

**Step 2: Determine main developer per file**
For each file, the author with the highest contribution (additions or commits) is the main developer. Calculate their ownership percentage: `main_author_contributions / total_contributions × 100`.

**Step 3: Identify bus-factor risks**
Files where the main developer accounts for >80% of contributions with no significant second contributor.

**Step 4: Cluster by developer**
Group files that share the same main developer to reveal expertise domains.

**Step 5: Present results**
```
## Knowledge Map

[Ethics warning from SKILL.md]

### Main Developers (top files by activity)
| File | Main Developer | Ownership % | 2nd Developer | 2nd % |
|------|---------------|-------------|---------------|-------|

### Expertise Domains
[Group files by main developer — shows who knows what]

### Bus Factor Risks
[Files where one person holds >80% of recent knowledge with no clear backup]
```

### Interpretation Guide
- A file with a clear main developer (>70%) and a strong second developer = healthy ownership
- A file with no developer above 30% = diffused knowledge, high coordination cost
- Clusters of files under one developer = that person's departure would create a knowledge gap — see `/xray knowledge-loss`
- Check for `.mailmap` file in repo root — multiple aliases for the same developer bias results

---

## fragmentation

### Purpose
Measure developer diffusion using the fractal value metric. Files with many minor contributors and no clear owner have higher defect risk and coordination overhead. Cross-referencing fragmentation with hotspots reveals the highest-risk code in the system.

### Procedure

**Step 1: For each of the top 50 hotspot files, get author commit counts**
```bash
git shortlog -sn --no-merges --after="<date>" -- "<file>"
```

**Step 2: Calculate fractal value**
For each file with its list of (author, commit_count) pairs:
```
total = sum of all commit counts
fractal_value = 1 - sum((commit_count_i / total)^2 for each author i)
```
- 0.0 = single author (no fragmentation)
- Approaching 1.0 = many equal contributors (high fragmentation)

**Step 3: Run hotspot analysis**
Use the change frequency data from the hotspots procedure to identify which fragmented files are also hotspots.

**Step 4: Create intersection set**
Files appearing in BOTH the top 20 hotspots AND the top 20 most fragmented files = **critical risk**.

**Step 5: Present results**
```
## Developer Fragmentation

[Ethics warning from SKILL.md]

### Most Fragmented Files
| File | Fractal Value | Authors | Top Author % | Hotspot? |
|------|--------------|---------|-------------|----------|

### CRITICAL: Hotspot + High Fragmentation
These files are frequently changed AND have no clear owner — highest coordination risk:
| File | Changes | Fractal Value | Authors |
|------|---------|--------------|---------|

### Recommendations
```

### Interpretation Guide
- Fractal value > 0.7 with > 5 authors = significant coordination bottleneck
- A hotspot with fractal value > 0.5 = the "immutable design" antipattern — everyone touches it, nobody owns it, nobody refactors it
- Consider introducing explicit ownership for high-fragmentation hotspots
- High fragmentation in test files = likely duplicate test patterns that each team implements independently

---

## rising

### Purpose
Early warning system. Detect files climbing the hotspot ranking — these are future maintenance problems developing right now. Catching them early means cheaper intervention.

### Procedure

**Step 1: Calculate current hotspot ranking**
Change frequencies for the last 6 months (standard hotspot analysis).

**Step 2: Calculate past hotspot ranking**
```bash
git log --format=format: --name-only --no-merges --after="<12_months_ago>" --before="<6_months_ago>" | egrep -v '^$' | sort | uniq -c | sort -rn | head -100
```

**Step 3: Compare rankings**
For each file in the current top 100, find its position in the past ranking. Calculate `rank_change = past_rank - current_rank`. A positive value means the file has climbed (become more active).

**Step 4: Identify rising hotspots**
- Files that climbed 10+ positions
- Files in the current top 50 that were NOT in the past top 100 at all (brand new hotspots)

**Step 5: Present results**
```
## Rising Hotspots (Early Warning)
Current period: <date> to today
Comparison period: <12mo_ago> to <6mo_ago>

### Files Climbing the Rankings
| File | Current Rank | Past Rank | Climb | Current Changes |
|------|-------------|-----------|-------|----------------|

### New Hotspots (absent from previous period)
| File | Current Rank | Changes |
|------|-------------|---------|

### Recommendations
```

### Interpretation Guide
- A rising hotspot may indicate: a new feature area under heavy development (expected) or code accumulating technical debt (investigate)
- Follow up on each rising hotspot with `/xray trend <file>` to determine if complexity is growing
- New hotspots that are already large files = highest risk — complexity was likely introduced upon creation
- Research shows problematic code smells are often introduced at creation, not accumulated later — catch them now

---

## knowledge-loss

### Purpose
Simulate the impact of a specific developer leaving. Identify files and components that would lose their main contributor. Cross-reference with hotspots and code age to assess severity.

### Procedure

**Step 1: Get the author's contributions**
```bash
git log --author="<author_name>" --format="%aN" --numstat --no-merges --after="<date>"
```
Sum additions per file for this author.

**Step 2: Get total contributions per file**
Use data from the knowledge map procedure. For each file the author touched, calculate:
```
author_percentage = author_additions / total_additions × 100
```

**Step 3: Classify risk per file**
- **HIGH:** Author contributed >80% of code (sole knowledge holder)
- **MEDIUM:** Author contributed 50–80% (primary contributor)
- **LOW:** Author contributed <50% (there is a capable backup)

**Step 4: Cross-reference with hotspots and age**
- A HIGH-risk file that is also a hotspot = CRITICAL (actively changed code losing its expert)
- A HIGH-risk file that is old and stable = lower concern (rarely needs modification)
- A HIGH-risk file that is mid-age = dangerous (we'll need to modify code we no longer understand)

**Step 5: Identify the next-best contributor**
For each HIGH-risk file, identify the second-highest contributor as the onboarding target.

**Step 6: Present results**
```
## Knowledge Loss Simulation: <author>

### Impact Summary
Files where <author> is main developer: N
- CRITICAL (high-risk + hotspot): N files
- HIGH risk (>80% ownership): N files
- MEDIUM risk (50-80%): N files
- LOW risk (<50%): N files

### Critical Files (high risk + actively changing)
| File | Author % | Next Developer | Next % | Changes | Age |
|------|----------|---------------|--------|---------|-----|

### High-Risk Files
| File | Author % | Next Developer | Next % |
|------|----------|---------------|--------|

### Recommendations
- Schedule knowledge transfer sessions for CRITICAL files
- Pair the departing developer with the next-best contributor on each HIGH-risk file
- Document domain-specific design decisions in code comments or ADRs
- Consider code walkthroughs for files with no clear backup contributor
```

---

## summary

### Purpose
Executive summary combining hotspots, coupling, and fragmentation into a single risk-prioritized report. Suitable for sharing with technical leads and nontechnical stakeholders.

### Procedure

**Step 1: Run hotspot analysis** (top 20)
Follow the `hotspots` procedure.

**Step 2: Run coupling analysis**
Follow the `coupling` procedure. Keep only surprising coupling.

**Step 3: Run fragmentation analysis** (top 20)
Follow the `fragmentation` procedure.

**Step 4: Cross-reference**
For each file in the top 20 hotspots, check if it also appears in:
- The coupling results (is it coupled to other files?)
- The top 20 fragmentation results (is it highly fragmented?)

Assign a composite risk level:
- **CRITICAL:** Hotspot + fragmented + coupled
- **HIGH:** Hotspot + fragmented, OR hotspot + coupled
- **MEDIUM:** Hotspot only
- **LOW:** Appears only in coupling or fragmentation but not hotspots

**Step 5: Present results**
```
## Behavioral Code Analysis Summary

### System Health Overview
Total commits: N | Active files: N | Active authors: N
Top hotspot concentration: X% of commits touch the top 10 files

### Priority Files (risk-ranked)
| Rank | File | Risk | Changes | LOC | Fragmentation | Coupled To |
|------|------|------|---------|-----|---------------|------------|

### Key Findings
1. [Most critical hotspot — why it matters and what to do]
2. [Most surprising coupling — what dependency it reveals]
3. [Highest fragmentation risk — coordination bottleneck identified]

### Top 5 Recommended Actions
1. [Highest-ROI refactoring target with specific technique]
2. [Ownership assignment recommendation]
3. [Code review focus area]
4. [Coupling to investigate or break]
5. [Architecture consideration]

### For Management
- X% of development effort concentrates on N files — improvements here have outsized impact
- [Key coordination bottleneck in plain language]
- [Risk assessment for upcoming work]
```

---

## Common Utilities

### Computing the --after date
```bash
date -v-6m +%Y-%m-%d    # macOS: 6 months ago
date -d "6 months ago" +%Y-%m-%d    # Linux: 6 months ago
```
Try the macOS variant first. If it fails, try Linux. Adjust `-6m` / `"6 months ago"` based on the analysis window.

### Checking for .mailmap
```bash
test -f .mailmap && echo "Found .mailmap - author aliases will be resolved" || echo "No .mailmap found - watch for duplicate author aliases"
```

### Scoping to a subdirectory
When the user provides `--path=<dir>`, append `-- <dir>/` to all `git log` commands. This restricts analysis to that subtree.

### Handling large repos
If `git log` commands take more than 30 seconds, suggest the user scope to a subdirectory with `--path=`. For hotspot and coupling analyses, limit initial git log output with `head -5000` to prevent excessive processing time.

### Using cloc instead of wc -l
When `cloc` is available, use it for LOC measurements:
```bash
cloc --csv --quiet --report-file=/tmp/xray_cloc.csv .
```
Parse the CSV to get per-language, per-file line counts (code column, excluding comments and blanks). This gives much more accurate hotspot scores since it excludes comments and blank lines.

For a single file:
```bash
cloc --csv --quiet "<file>" | tail -1 | cut -d, -f5
```

### Using Code Maat
When the Code Maat JAR is available, it can perform coupling and author analyses on a git log export:

**Export git log in Code Maat format:**
```bash
git log --all --numstat --date=short --pretty=format:'--%h--%ad--%aN' --no-renames --no-merges --after="<date>" > /tmp/xray_git.log
```

**Run coupling analysis:**
```bash
java -jar code-maat-standalone.jar -l /tmp/xray_git.log -c git2 -a coupling
```

**Run author analysis:**
```bash
java -jar code-maat-standalone.jar -l /tmp/xray_git.log -c git2 -a main-dev
```

**Run entity effort (fragmentation):**
```bash
java -jar code-maat-standalone.jar -l /tmp/xray_git.log -c git2 -a entity-effort
```

Code Maat outputs CSV which is easier to parse than raw git output. Use it when available, but the git+shell fallback produces equivalent results.

---

## visualize

### Purpose
Generate interactive HTML visualizations using D3.js that can be opened in a browser. These visualizations match the enclosure diagrams and hierarchical edge bundles used throughout "Software Design X-Rays" and let you see the whole codebase at a glance — something tables cannot do.

### Subcommands

#### visualize hotspots

**Purpose:** Enclosure diagram — nested circles following folder hierarchy. Circle size represents lines of code, color intensity represents change frequency (deeper red = more commits).

**Procedure:**

**Step 1: Gather data**
Run the standard hotspots procedure to get change frequencies and LOC for all files.

**Step 2: Build hierarchical JSON**
Transform the flat file list into a nested tree structure matching the folder hierarchy:
```json
{
  "name": "root",
  "children": [
    {
      "name": "src",
      "children": [
        { "name": "core",
          "children": [
            { "name": "engine.ts", "loc": 1200, "changes": 85, "score": 0.92 }
          ]
        }
      ]
    }
  ]
}
```
Each leaf node is a file with `loc`, `changes`, and `score` fields. Each directory node has a `children` array.

**Step 3: Generate HTML file**
Write the following self-contained HTML to `/tmp/xray-hotspots.html`:

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>X-Ray: Hotspot Enclosure Diagram</title>
<style>
  body { margin: 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: #0d1117; color: #c9d1d9; }
  .header { padding: 16px 24px; background: #161b22; border-bottom: 1px solid #30363d; }
  .header h1 { margin: 0; font-size: 18px; font-weight: 600; }
  .header p { margin: 4px 0 0; font-size: 13px; color: #8b949e; }
  svg { display: block; margin: 0 auto; }
  circle { cursor: pointer; }
  .tooltip { position: absolute; padding: 8px 12px; background: #1c2128; border: 1px solid #30363d; border-radius: 6px; font-size: 12px; pointer-events: none; opacity: 0; transition: opacity 0.15s; }
  .legend { position: fixed; bottom: 16px; right: 16px; background: #161b22; border: 1px solid #30363d; border-radius: 6px; padding: 12px; font-size: 12px; }
  .legend-row { display: flex; align-items: center; gap: 8px; margin: 4px 0; }
  .legend-swatch { width: 14px; height: 14px; border-radius: 3px; }
</style>
</head>
<body>
<div class="header">
  <h1>X-Ray: Hotspot Map</h1>
  <p>REPO_NAME — ANALYSIS_PERIOD — Circle size = lines of code, color = change frequency (click to zoom)</p>
</div>
<div class="tooltip" id="tooltip"></div>
<div class="legend">
  <div class="legend-row"><div class="legend-swatch" style="background:#1a1a2e"></div> Low activity</div>
  <div class="legend-row"><div class="legend-swatch" style="background:#6b2020"></div> Medium activity</div>
  <div class="legend-row"><div class="legend-swatch" style="background:#e63946"></div> High activity (hotspot)</div>
  <div class="legend-row" style="margin-top:8px; color:#8b949e">Click circles to zoom in/out</div>
</div>
<script src="https://d3js.org/d3.v7.min.js"></script>
<script>
const data = DATA_JSON;

const width = Math.min(window.innerWidth, 1200);
const height = window.innerHeight - 60;

const color = d3.scaleSequential([0, 1], t => d3.interpolateRgb("#1a1a2e", "#e63946")(t));
const dirColor = "#161b22";

const pack = d3.pack().size([width, height]).padding(3);
const root = pack(d3.hierarchy(data).sum(d => d.loc || 1).sort((a, b) => (b.data.loc || 0) - (a.data.loc || 0)));

let focus = root;
let view;

const svg = d3.select("body").append("svg")
  .attr("width", width).attr("height", height)
  .attr("viewBox", `-${width/2} -${height/2} ${width} ${height}`)
  .style("cursor", "pointer")
  .on("click", () => zoom(root));

const node = svg.append("g").selectAll("circle")
  .data(root.descendants().slice(1))
  .join("circle")
  .attr("fill", d => d.children ? dirColor : color(d.data.score || 0))
  .attr("stroke", d => d.children ? "#30363d" : "none")
  .attr("stroke-width", d => d.children ? 1 : 0)
  .attr("pointer-events", d => !d.children ? "all" : null)
  .on("mouseover", function(event, d) {
    d3.select(this).attr("stroke", "#58a6ff").attr("stroke-width", 2);
    const tip = d.children
      ? `<strong>${d.data.name}/</strong><br>${d.descendants().filter(n=>!n.children).length} files`
      : `<strong>${d.data.name}</strong><br>LOC: ${d.data.loc}<br>Changes: ${d.data.changes}<br>Score: ${(d.data.score||0).toFixed(2)}`;
    d3.select("#tooltip").html(tip).style("opacity",1)
      .style("left",(event.pageX+12)+"px").style("top",(event.pageY-12)+"px");
  })
  .on("mousemove", function(event) {
    d3.select("#tooltip").style("left",(event.pageX+12)+"px").style("top",(event.pageY-12)+"px");
  })
  .on("mouseout", function(event, d) {
    d3.select(this).attr("stroke", d.children ? "#30363d" : "none").attr("stroke-width", d.children ? 1 : 0);
    d3.select("#tooltip").style("opacity",0);
  })
  .on("click", (event, d) => { if (focus !== d) { zoom(d); event.stopPropagation(); } });

const label = svg.append("g")
  .style("font", "11px sans-serif").attr("fill", "#c9d1d9")
  .attr("pointer-events", "none").attr("text-anchor", "middle")
  .selectAll("text").data(root.descendants())
  .join("text").style("fill-opacity", d => d.parent === root ? 1 : 0)
  .style("display", d => d.parent === root ? "inline" : "none")
  .text(d => d.data.name);

zoomTo([root.x, root.y, root.r * 2]);

function zoomTo(v) {
  const k = width / v[2];
  view = v;
  label.attr("transform", d => `translate(${(d.x - v[0]) * k},${(d.y - v[1]) * k})`);
  node.attr("transform", d => `translate(${(d.x - v[0]) * k},${(d.y - v[1]) * k})`);
  node.attr("r", d => d.r * k);
}

function zoom(d) {
  focus = d;
  const transition = svg.transition().duration(750).tween("zoom", () => {
    const i = d3.interpolateZoom(view, [focus.x, focus.y, focus.r * 2]);
    return t => zoomTo(i(t));
  });
  label.filter(function(d) { return d.parent === focus || this.style.display === "inline"; })
    .transition(transition).style("fill-opacity", d => d.parent === focus ? 1 : 0)
    .on("start", function(d) { if (d.parent === focus) this.style.display = "inline"; })
    .on("end", function(d) { if (d.parent !== focus) this.style.display = "none"; });
}
</script>
</body>
</html>
```

**Step 4: Inject data**
Replace `DATA_JSON` with the actual JSON tree from Step 2. Replace `REPO_NAME` and `ANALYSIS_PERIOD` with actual values.

**Step 5: Open in browser**
```bash
open /tmp/xray-hotspots.html      # macOS
xdg-open /tmp/xray-hotspots.html  # Linux
```
Try macOS first, fall back to Linux.

Report to the user: "Hotspot visualization saved to `/tmp/xray-hotspots.html` and opened in your browser."

---

#### visualize coupling

**Purpose:** Hierarchical edge bundle — files arranged in a circle grouped by directory, with curved links showing change coupling relationships between them. Hover over a file to highlight its coupled dependencies.

**Procedure:**

**Step 1: Gather data**
Run the standard coupling procedure to get all coupled file pairs with their degree.

**Step 2: Build node and link data**
Nodes: each file that appears in any coupling pair, with its directory path for hierarchical grouping.
Links: each coupling pair with its degree as weight.

```json
{
  "nodes": [
    { "id": "src/core/engine.ts", "group": "src/core" },
    { "id": "src/api/handler.ts", "group": "src/api" },
    { "id": "test/core/engine.test.ts", "group": "test/core" }
  ],
  "links": [
    { "source": "src/core/engine.ts", "target": "test/core/engine.test.ts", "degree": 85, "cochanges": 34, "surprising": false },
    { "source": "src/core/engine.ts", "target": "src/api/handler.ts", "degree": 62, "cochanges": 18, "surprising": true }
  ]
}
```

**Step 3: Generate HTML file**
Write the following self-contained HTML to `/tmp/xray-coupling.html`:

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>X-Ray: Change Coupling</title>
<style>
  body { margin: 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: #0d1117; color: #c9d1d9; }
  .header { padding: 16px 24px; background: #161b22; border-bottom: 1px solid #30363d; }
  .header h1 { margin: 0; font-size: 18px; font-weight: 600; }
  .header p { margin: 4px 0 0; font-size: 13px; color: #8b949e; }
  .node text { font: 11px sans-serif; fill: #8b949e; }
  .node text.highlight { fill: #f0f6fc; font-weight: 600; }
  .link { fill: none; stroke-opacity: 0.15; }
  .link.expected { stroke: #388bfd; }
  .link.surprising { stroke: #e63946; }
  .link.highlight { stroke-opacity: 0.8; }
  .node circle { fill: #30363d; stroke: none; }
  .node circle.highlight { fill: #58a6ff; }
  .tooltip { position: absolute; padding: 8px 12px; background: #1c2128; border: 1px solid #30363d; border-radius: 6px; font-size: 12px; pointer-events: none; opacity: 0; transition: opacity 0.15s; }
  .legend { position: fixed; bottom: 16px; right: 16px; background: #161b22; border: 1px solid #30363d; border-radius: 6px; padding: 12px; font-size: 12px; }
  .legend-row { display: flex; align-items: center; gap: 8px; margin: 4px 0; }
  .legend-swatch { width: 24px; height: 3px; border-radius: 2px; }
</style>
</head>
<body>
<div class="header">
  <h1>X-Ray: Change Coupling</h1>
  <p>REPO_NAME — ANALYSIS_PERIOD — Hover over a file to see its coupling dependencies</p>
</div>
<div class="tooltip" id="tooltip"></div>
<div class="legend">
  <div class="legend-row"><div class="legend-swatch" style="background:#e63946"></div> Surprising coupling</div>
  <div class="legend-row"><div class="legend-swatch" style="background:#388bfd"></div> Expected coupling</div>
  <div class="legend-row" style="margin-top:8px; color:#8b949e">Hover files to highlight</div>
</div>
<script src="https://d3js.org/d3.v7.min.js"></script>
<script>
const data = DATA_JSON;
const width = Math.min(window.innerWidth, 1100);
const height = window.innerHeight - 60;
const radius = Math.min(width, height) / 2 - 120;

// Sort nodes by group for clustering
const nodes = data.nodes.sort((a, b) => d3.ascending(a.group, b.group));
const nodeMap = new Map(nodes.map((n, i) => [n.id, i]));
const n = nodes.length;

const svg = d3.select("body").append("svg")
  .attr("width", width).attr("height", height)
  .append("g").attr("transform", `translate(${width/2},${height/2})`);

// Position nodes in a circle
nodes.forEach((node, i) => {
  const angle = (2 * Math.PI * i) / n - Math.PI / 2;
  node.x = Math.cos(angle) * radius;
  node.y = Math.sin(angle) * radius;
  node.angle = angle;
});

// Draw links as quadratic curves through center
const link = svg.append("g").selectAll("path")
  .data(data.links).join("path")
  .attr("class", d => `link ${d.surprising ? "surprising" : "expected"}`)
  .attr("stroke-width", d => Math.max(1, d.degree / 30))
  .attr("d", d => {
    const si = nodeMap.get(d.source), ti = nodeMap.get(d.target);
    if (si === undefined || ti === undefined) return "";
    const s = nodes[si], t = nodes[ti];
    return `M${s.x},${s.y}Q0,0 ${t.x},${t.y}`;
  });

// Draw nodes
const nodeG = svg.append("g").selectAll("g")
  .data(nodes).join("g").attr("class", "node")
  .attr("transform", d => `translate(${d.x},${d.y})`);

nodeG.append("circle").attr("r", 4);

nodeG.append("text")
  .attr("dx", d => d.angle > Math.PI/2 && d.angle < 3*Math.PI/2 ? -8 : 8)
  .attr("dy", "0.35em")
  .attr("text-anchor", d => d.angle > Math.PI/2 && d.angle < 3*Math.PI/2 ? "end" : "start")
  .attr("transform", d => {
    const deg = d.angle * 180 / Math.PI;
    const flip = deg > 90 && deg < 270;
    return `rotate(${flip ? deg + 180 : deg})`;
  })
  .text(d => d.id.split("/").pop());

// Hover interactions
nodeG.on("mouseover", function(event, d) {
  const connected = new Set();
  connected.add(d.id);
  const tips = [];
  link.classed("highlight", l => {
    if (l.source === d.id || l.target === d.id) {
      connected.add(l.source); connected.add(l.target);
      tips.push(`${l.source === d.id ? l.target : l.source}: ${l.degree}% (${l.cochanges} co-changes)${l.surprising ? " [surprising]" : ""}`);
      return true;
    }
    return false;
  });
  nodeG.select("circle").classed("highlight", n => connected.has(n.id));
  nodeG.select("text").classed("highlight", n => connected.has(n.id));
  d3.select("#tooltip")
    .html(`<strong>${d.id}</strong><br>${tips.join("<br>")}`)
    .style("opacity", 1)
    .style("left", (event.pageX + 12) + "px")
    .style("top", (event.pageY - 12) + "px");
})
.on("mousemove", function(event) {
  d3.select("#tooltip").style("left",(event.pageX+12)+"px").style("top",(event.pageY-12)+"px");
})
.on("mouseout", function() {
  link.classed("highlight", false);
  nodeG.select("circle").classed("highlight", false);
  nodeG.select("text").classed("highlight", false);
  d3.select("#tooltip").style("opacity", 0);
});
</script>
</body>
</html>
```

**Step 4: Inject data**
Replace `DATA_JSON` with the actual node/link JSON from Step 2. Replace `REPO_NAME` and `ANALYSIS_PERIOD`.

**Step 5: Open in browser**
```bash
open /tmp/xray-coupling.html      # macOS
xdg-open /tmp/xray-coupling.html  # Linux
```

Report to the user: "Coupling visualization saved to `/tmp/xray-coupling.html` and opened in your browser."
