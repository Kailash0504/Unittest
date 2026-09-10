# xUnit Test Audit Report

A single-file dashboard that audits the xUnit test suite of `Casepoint.API.Core.Test` and
reports on it live from TFS. Open `index.html` and connect with a PAT: the pipeline and solution
come from `appsettings.json`, and the report builds itself from that pipeline's latest completed
run — there is no build step, no server, and no data baked into the file.

## Scope: the whole solution

The build covers a whole solution, so the report does too. `solution.name` in `appsettings.json`
names the `.sln` **by file name**, and the report finds where it lives. It then reads that
solution's project list, checks each `.csproj` for an `xunit` reference, and includes **every**
test project it finds — not one hardcoded path.

Identifying the solution by name rather than by full path means a moved or mistyped path cannot
break the report. `solution.path` is an optional hint: when it resolves, the solution is fetched
directly and no repository listing happens. When it does not, the report searches the repository
for the file name, uses what it finds, and says in the banner that the hint needs updating — so a
stale path is visible and self-correcting rather than a silent 404.

xunit references are what identify a test project, rather than its name, so a test project named
anything at all is found, and a helper library called `*.Testing.*` with no xunit reference is
correctly left out. The header shows the solution and how many of its projects qualify, e.g.
`Scope: 2 of 4 projects use xunit`.

Test project is the top level throughout: the first panel is a per-test-project breakdown, the File
Inventory carries a **Test Project** column, and the KPIs total across all of them. Source is
downloaded as one zip per test project, and commit history runs one paged query per test-project
path, merged on commit id so a commit touching two of them counts once.

If no solution is configured, or none matching the name exists on the branch, the report falls
back to `fallbackTestProjectPath` and scans that single project instead, with the reason shown in
the banner and the header reading `Scope: single test project`.

## Test files vs support files

Roughly half the `.cs` files in a test project hold no tests — `*Setup.cs` files that wire up
mocks, `GlobalTestHelper.cs`, `*Helpers.cs`, `PrivateAccess.cs`. They are counted separately:

- **Test Files** and the **File Inventory** cover only files holding at least one `[Fact]` or
  `[Theory]`. The KPI sub-text and the panel note both state how many support files were withheld.
- **Total LOC** and the **average LOC per file** still cover *every* scanned file. Support files are
  real maintained test code, so excluding their lines would understate the test codebase — and
  because the average feeds the sprint **Est. LOC Activity**, dropping them would silently inflate
  every sprint estimate.
- The per-project breakdown shows test files in its column, with the full scanned count on hover.

A file is classified by **its own test count, never by its name**. A name filter would be wrong on
this repository: `GlobalTestHelper.cs` appears once per test project, matches any `*Test*` pattern,
and contains no tests at all. `CloudTestHelper.cs` and `IdpGlobalTestHelper.cs` are the same.

## Connecting

1. Open `index.html` (GitHub Pages serves it at the repository root URL).
2. Leave **TFS Server URL** as `https://tfs.casepoint.in/tfs/Casepoint` unless you are pointing at
   another collection.
3. Paste a **Personal Access Token**. Projects load as soon as the field loses focus.
4. Press **Connect & Generate Report**.

There is no pipeline or solution to choose: both come from `appsettings.json`, and the project
named there is ticked for you, so connecting is a single click. What will be generated is listed
on the form under **Reporting on** before you press it.

The URL and PAT are kept in `localStorage` under the `unitTestAudit_` prefix so a reload
reconnects without retyping them. The report also accepts a handoff link of the form
`?tfsUrl=…&pat=<base64>&project=…&autoLoad=1`, which is what the Sprint Requirements Tracker
opens it with.

Use **Refresh** in the toolbar to rebuild the report — note that this re-resolves the pipeline,
so it will pick up a newer successful build if one has completed since. **Reconnect** returns
to the connection panel.

The PAT needs read access to **Code**, **Build**, **Test Management**, and **Project and Team**.

## How the data is assembled

The selected pipeline's latest completed build, succeeded or partially succeeded, is the anchor —
its branch and commit determine what gets scanned, so every panel describes the same point in
history.

| Panel | Source |
| --- | --- |
| Total tests, Facts, Theories, Skipped, Test Files, Total LOC | `.cs` sources at the **tip of the build's branch**, downloaded as one zip and parsed in the browser |
| Module / Folder Breakdown, Test File Inventory | the same source scan, grouped by first folder segment |
| Pipeline Pass Rate, Pipeline Test Run Results | test runs published against the build |
| Contributor Activity, Recent Commits | every commit touching the test project path on the build's branch |
| Sprint-wise Comparison | the same commits, bucketed by the project's real iterations |
| Code Coverage finding | coverage published by the build, when the pipeline publishes any |
| Compliance & Governance Findings | derived from the sources above |

Every commit query is scoped to `testProjectPath`, so the contributor and commit panels show only
people and changes that touched the test project — the same view as the TFS history page for that
path — rather than all branch activity. The commit walk is paged, so it covers the path's whole
history rather than a fixed window.

**Everything is current except the pipeline results.** The pipeline run supplies the repository and
branch, but the source is then read at that branch's tip and the commit history is aggregated over
the path's entire history up to today, so the counts, contributor totals and sprint buckets all
describe the project as it stands now. The only figures tied to the build itself are the pass rate
and the Pipeline Test Run Results panel — inherently so, since they come from that run. The header
shows the build's date next to its number, so a stale run is visible at a glance.

Build timings are reported the way TFS reports them. A build is listed under the time it
**started**, so that is what "Started" means in both the header and the run table — reporting the
run's completion instead put the report 16 minutes ahead of the pipeline page. **Duration** is
start to finish, excluding queue time, because waiting for an agent is not part of the build; it is
the same figure TFS shows beside a run. The header carries start, duration and result together,
each recent-build pill carries its own duration so a slow run stands out, and hovering a run row
gives the test run's *own* start, finish and duration — which differ from the build's, since a run
begins after the build has and can finish before it ends.

`recentCommitCount` limits the Recent Commits panel to that many **rows**, not to a period; it does
not affect any other figure.

Panels fail independently: if the source scan is refused, the module and file panels show the error
and their KPIs blank out while the pipeline, contributor and sprint panels still render.

Facts and Theories are counted from `[Fact]` and `[Theory]` attributes in the source rather than
from test results, because the test result payload cannot distinguish them and carries no LOC.
Coverage cannot substitute for the scan either: it reports covered and uncovered lines per
compiled module, with no notion of attributes, per-file rows, or lines of test code.

## Sprints

The sprint table uses the **project's own iterations**, read from
`wit/classificationnodes/iterations`. Each row is a real iteration path — `PI2026-3\Sprint 2` —
with the full path as the cell's tooltip and the iteration's own start and finish dates as its
range. Only dated **leaf** iterations count: a parent's dates span its children, so including both
would attribute every commit twice. Iterations that do not overlap the commit window are dropped,
so the table stays bounded however long the project's iteration tree is.

Where iterations from different teams overlap, a commit is attributed to the **narrowest** matching
iteration, on the grounds that it is the most specific. Commits that fall in a gap between
iterations are gathered into a single `Outside any iteration` row, which appears only when such
commits exist. Because the leaf name repeats across parents, a chart label that would be ambiguous
falls back to the fuller path.

If the project has no dated iterations covering the window — or the iterations call fails, which is
not fatal — the report falls back to 14-day buckets labelled `SP-01` onward, starting from the
first commit that ever touched the test project. Either way the window ends today, and the baseline
figure in the reconciliation card is zero by construction: there is no test code before the commit
that created it.

## Pass rate

The pass rate is `passed / (passed + failed)` — tests that never ran (skipped or not executed) are
excluded from the denominator, so the figure reflects only tests that actually executed. The count
of tests that did not run is reported alongside it in the compliance finding, and the `Skipped` KPI
counts `Skip=` attributes found in the source, which is a different measure: what the code declares
skipped, rather than what the run reported as not executed.

The **Pipeline Pass Rate** card spans two grid slots and carries the whole run beside the rate —
Total, Passed, Failed, and Not Run when any test did not execute — with the build number underneath.
Because the rate divides by executed rather than by total, the sub-text names its denominator
(`7,802 of 7,835 executed`) and adds `(N total)` whenever the two diverge, so the percentage is
never read against the wrong number. Total, Passed, Failed and Not Run always reconcile.

## Configuration

`appsettings.json` decides what gets reported. Despite the extension it is loaded as a
`<script>`, so it must stay valid JavaScript - that is the convention the sister dashboards use,
and it is why an editor may warn that "comments are not permitted in JSON".

| Key | Meaning |
| --- | --- |
| `project` | team project holding the pipeline and solution; ticked for you on the form |
| `pipeline.id` | build definition id. Wins when set, because it survives a rename |
| `pipeline.name` | build definition name, matched exactly (case-insensitive) when no id is set |
| `solution.name` | file name of the `.sln` to report on; every project in it referencing xunit is included |
| `solution.path` | optional fast hint at where that file lives; falls back to a search when wrong |
| `fallbackTestProjectPath` | used only when `solution.path` is empty or unreadable |
| `users` | access list; see below |

The pipeline and the solution exist only inside `project`, so connecting with a different project
selected is refused with an explanation naming the project to pick, rather than failing later with
a confusing "no pipeline matches". Retarget by changing all three together, or set `project` to
`""` to lift the check and choose by hand.

If `pipeline.name` matches nothing, any definition whose name looks like a unit test pipeline is
used as a last resort, so a fresh clone still produces a report.

Remaining constants live in the `CONFIG` block near the top of the script in `index.html`:

| Key | Meaning |
| --- | --- |
| `testProjectPath` | repository path scanned for tests and used to scope every commit query |
| `sprintLengthDays` | bucket width for the fallback scheme, when the project has no dated iterations |
| `sprintOriginFallback` | fallback bucket start, used only when the path has no commit history |
| `commitPageSize`, `maxCommitPages` | page size and page cap for the commit history walk |
| `recentCommitCount` | rows in the Recent Commits panel |
| `passRateThreshold` | pass rate below which compliance raises a flag |

## What is measured and what is estimated

Everything in the report is exact except two clearly-marked things, both in the file-churn columns.

**Exact.** Test Files, Total LOC, Facts, Theories and Skipped, plus every module and file row —
counted from the actual `.cs` files at the branch tip. Total, Passed, Failed and the pass rate —
from the published test runs. Coverage — from the coverage API. Commit counts, contributor counts,
dates, the first-commit date and iteration membership — from the commit history, scoped to
`testProjectPath`.

**Not exact, and labelled `*` in the tables.** The `Files Added / Modified / Deleted` columns are
each commit's own `changeCounts`, which TFS reports **per commit, not per path**: `itemPath` selects
*which* commits come back, but their counts cover every file in that commit, repo-wide. A merge
touching files elsewhere in the repository inflates them. `Est. LOC Activity` is then
`(added + modified) × avg LOC per file`, so it inherits that error and compounds it — treat it as a
relative activity signal between sprints, never as a line count. Both tables carry a footnote
saying so, and the reconciliation card lists all four reasons it overstates.

Making the file columns exact would need `/commits/{id}/changes` per commit — roughly 555 extra
calls per load — which is why they are labelled rather than recomputed.

**Coverage** comes from `test/codecoverage?buildId=` for the anchor build, aggregated over all
modules as `covered / (covered + partiallyCovered + notCovered)`. Partially-covered lines count as
uncovered, which is the conservative reading. It reflects the assemblies the run instrumented —
product code — not the test project's own lines, and it appears only if the pipeline publishes it.

Two smaller limits worth knowing: attribute counts come from regular expressions over the source,
so a commented-out `[Fact]` would still be counted; and contributors are grouped by email address,
so one person committing under two addresses counts twice.

## Caveats

- **Estimated LOC is an activity metric, not a line count.** Per-sprint LOC is
  `(files added + files modified) × average LOC per file`, so a file edited in several sprints is
  counted more than once and a branch-sync merge inflates a single bucket. The reconciliation
  card in the sprint panel spells this out; `Total LOC` is the authoritative figure.
- **The anchor is the newest completed build that published test runs.** Result is irrelevant —
  succeeded, partially succeeded and failed all qualify — because filtering by result would pin the
  report to the last run with no failing test and show a permanent 100% pass rate. Only cancelled
  runs are skipped outright (`ignoredBuildResults`); they stop before producing anything.
  Since the pass rate can only come from published test runs, builds are probed newest-first (up to
  `maxBuildProbes`) until one has usable runs. A run counts as usable when it has tests and its
  state is in `usableRunStates` — `Completed` **and** `NeedsInvestigation`, the latter being a
  finished run whose tests failed, which must count or the pass rate skews green again.
- The pipeline panel lists the last `recentBuildCount` completed runs with their result and date,
  marking the one being shown and labelling any that were passed over for having no test results.
  If nothing in the probe window published results, the newest build is shown for reference and the
  panel says how many were checked.
- **The pass rate is floored, not rounded**, so 6,290 of 6,300 reads `99%`. Only a genuinely
  perfect run shows `100%`.
- **Cross-origin access to TFS must be permitted** for the browser to read the API, including the
  binary zip response used by the source scan.
- **The report needs no internet access**, only a route to TFS — see [No dependencies](#no-dependencies).

## Theme

The toolbar's first button toggles light and dark, and the choice is remembered in
`localStorage` under `unitTestAudit_theme`. Both palettes are defined as variables on `:root` and
`body.light-theme`, and the SVG charts read their grid, tick and label colours from those variables
at render time, so they follow the theme rather than staying dark.

## Export PDF

The toolbar's **Export PDF** button opens the browser's print dialog — choose *Save as PDF*
(Ctrl+P does the same). There is no library involved: a `@media print` block restyles the report
for paper, so the output keeps **selectable text and vector charts** rather than being a screenshot.

Printing forces the light palette whatever the screen theme is, hides the connect forms, toolbar
and API counter, expands every collapsed panel (a collapsed one would print as an empty strip),
lifts the collapse height limit, and defaults to A4 landscape for the wide tables.

Fitting the wide tables onto paper needs three things working together: nothing in the panel
chain may clip (`.cp`, `.table-wrap` and `.kpi-card` all clip on screen, and a single one of them
left clipping silently cuts off the right-hand columns); type and padding are reduced so all
columns fit the page width; and only the leading identifier columns may break mid-token, because
letting a numeric column shrink that far renders a value as "1,035,3 / 9". Page breaks are avoided
inside rows, KPI cards and compliance items but *not* inside a panel — a panel taller than a page
cannot honour the request, and asking anyway just forces a blank page ahead of it. Panel open/closed state is restored
afterwards, and because the behaviour is bound to the `beforeprint`/`afterprint` events, Ctrl+P
behaves exactly like the button.

The sister dashboards take a different route — `html2canvas` renders a hidden iframe to a canvas
and `jsPDF` embeds it as a PNG — which needs both libraries from a CDN and produces an image, so
the text cannot be selected or searched. This report avoids both problems.

## Access list (optional)

The `users` array in `appsettings.json` works the same way as `allowed_users.json` does in the
sister dashboards. On PAT entry the report calls
`_apis/connectionData`, shows who the token belongs to, and compares that display name against the
list.

- `users: []` (as shipped) - no gate; anyone with a valid PAT may use the report
- `users: [ ... ]` - only the listed people may load it

Names are obfuscated with the same XOR+base64 cipher and key as the sister dashboards, so a value
encrypted by the Tracker's "Encrypt Username" tab works here unchanged.

**This is a soft gate, not security.** The page is a static file: anyone can read the list, read the
cipher key, or bypass the check with dev tools. It makes the intended audience explicit and keeps
the report tidy. The real protection is the PAT — without a valid one, no TFS data can be read.

## No dependencies

`index.html` loads nothing from anywhere except your TFS server. There is no CDN, no `vendor/`
folder and no build step, because the network this runs on does not reach cdnjs, jsdelivr or
unpkg. The two things a library would normally provide are built in:

- **Unzipping the source archive** uses the browser's own `DecompressionStream('deflate-raw')` with
  a small ZIP central-directory reader, in place of JSZip.
- **The four sprint graphs** are generated as inline SVG, in place of Chart.js.

This needs a Chromium- or Firefox-based browser from 2022 or later (`DecompressionStream`). On
anything older the source-scan panels explain that and the rest of the report still loads. The file
is also pure ASCII, so it cannot be mangled by a tool that guesses the wrong encoding.

## Local development

Browsers block `fetch` from `file://` origins, so serve the directory over HTTP rather than
double-clicking the file:

```sh
npx serve .
```
