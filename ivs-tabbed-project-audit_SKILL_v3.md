---
name: ivs-tabbed-project-audit
description: >
  Generates the IVS Tabbed Project Audit — a data-integrity audit that uses a
  live CMC Portfolio Dashboard HTML export as the BASELINE and reconciles it,
  tab by tab and project by project, against raw Smartsheet source
  documentation. Output is a single HTML file with four top-level tabs
  matching the dashboard's own navigation — Programs, Mfg Roadmap, Risk
  Register, plus a fourth "Other Tabs & Summary" tab covering Ask the
  Portfolio and the Discrepancy Log — each tab organized strictly by Project
  Name with a filter/search bar and collapse/expand controls. Every data
  point is marked [VERIFIED] or [DISCREPANCY: detail] against source; tabs
  or fields that don't exist on either side are marked [N/A] with an
  explanation, never fabricated. Person/name fields are matched by name only
  (email suffixes, delimiter variance, and wrapping quotes are normalized
  away, not treated as mismatches) — see LAW E. By default, only reports on
  data the dashboard actually displays — source-only rows the dashboard
  never rendered are dropped, not shown as blank-vs-full comparisons — see
  LAW F.
  Requires: an uploaded dashboard HTML export (from the cmc-portfolio-dashboard
  skill) + live Smartsheet pulls of S1 Project Intake (2708701472313220),
  R4 Risk Report (6898181806706564), R5 Project Team (4594364127858564),
  R6 Schedule Detail (1032442042339204). R2/R3 aggregated reports are
  consulted but historically lack a Sheet Name column — resolvable via
  per-sheet_id calls, see "Resolving R2/R3 Sheet Names".
  Trigger on: "IVS tabbed audit", "IVS project audit", "dashboard baseline
  audit", "audit the dashboard by tab", "audit the dashboard sub-tabs",
  "IVD tabbed audit" (common typo for IVS), or any request to reconcile a
  CMC Portfolio Dashboard export against raw source data organized by tab
  and project.
---

# IVS Tabbed Project Audit — Skill

This skill produces a single HTML deliverable auditing a **CMC Portfolio
Dashboard export** (an HTML file built by the `cmc-portfolio-dashboard`
skill) against **live raw Smartsheet source data**. It differs from the
`gc-ivs-integrity-report` skill: that skill audits the *Smartsheet reports
themselves* for internal consistency; this skill audits whether a
*downstream dashboard* still accurately reflects those reports. Both skills
should be run together for a complete picture — this one assumes the other's
LAW 1–8 as a baseline and adds dashboard-specific verification.

---

## THE FOUR LAWS OF THIS AUDIT (in addition to LAW 1–8 from gc-ivs-integrity-report)

### LAW A — The dashboard's own embedded data is the baseline, not a re-rendering of it
Extract `PROGRAMS`, `FLAT_RISKS`, `ALL_SCHED`, `STATIC_ANALYSIS`, `LENSES`
(and any other top-level `const NAME = [...]` / `{...}` declarations)
directly from the uploaded HTML file's embedded JavaScript via balanced
bracket parsing — see Extraction Method below. Never re-derive what the
dashboard "should" show from the source data first and compare that against
itself; that isn't an audit, it's a tautology. Always pull the dashboard's
actual rendered arrays first, independently of source pulls.

### LAW B — Never fabricate a sub-tab or field that doesn't exist
Before claiming a discrepancy on a given sub-tab, confirm the field actually
exists in the dashboard's data model (e.g. `grep` for the key across all
`PROGRAMS` entries). If it doesn't exist anywhere, the correct output is
`[N/A — tab/field does not exist in dashboard data model]`, not a fabricated
empty comparison and not a silent omission. This session's audit found two
requested sub-tabs (Decision, Supply Chain Node) that do not exist as
distinct fields — see Known Structural Gaps.

### LAW C — A blocked source does not mean the dashboard side goes unreported
When a source report can't be grouped by project this session (e.g. R2/R3
lack a Sheet Name column on the aggregated report — see Known Structural
Gaps), still render the dashboard's content for that sub-tab in full,
verbatim. Mark the comparison column `[UNVERIFIABLE — <reason>]` rather than
omitting the sub-tab or guessing at a match. The person auditing needs to
see what the dashboard claims even when it can't be checked this session.
**Important refinement:** "blocked" on the aggregated report does not mean
blocked entirely. R2/R3's per-project *underlying sheets* can still be
resolved individually — see "Resolving R2/R3 Sheet Names" below. Don't
default to [UNVERIFIABLE] without first checking whether the cheaper
aggregated-report path is truly the only path.

### LAW D — Structural/header rows are not source-comparable and must not be scored as discrepancies
Dashboard schedule/roadmap data often contains synthetic grouping rows
(`isHeader: true`, e.g. "Schedule", "High Level Clinical") that exist only to
organize the UI and were never literal rows in the source report. Detect
these via their own flag field before comparison and mark them
`[N/A — structural header row]`, not `[DISCREPANCY]`. Conflating the two
inflates the apparent error rate and buries genuine findings.

### LAW E — Match people by name, not by exact field string
The dashboard renders team assignments as plain names ("Chris Tarapata").
The raw R5 source stores them as "Chris Tarapata <chris.tarapata@lilly.com>"
— and inconsistently uses comma, semicolon, AND slash as multi-person
delimiters across different rows, sometimes with a literal pair of quote
characters wrapping the whole field as part of the text content (not just
display formatting). An exact string comparison between these will flag
the overwhelming majority of genuinely correct assignments as
[DISCREPANCY], which is worse than useless — it's actively misleading. The
email address, differing delimiters, and stray literal quotes are all
formatting variance on the source side, not evidence of a wrong person.
**Always normalize before comparing**: strip `<email>` suffixes, strip a
literal wrapping quote pair from the whole field, split on `[,;/]`, then
compare the resulting name sets case-insensitively. See "Person/Name
Matching" below for the reference implementation — this exact bug was
found and fixed mid-session and dropped the false-positive rate from what
looked like majority-mismatched down to 1.0% genuinely mismatched (30 of
3,022 team entries). Also treat `''`, `'na'`, `'n/a'`, `'<<update>>'`,
`'tbd'` as equivalent "unassigned" placeholders on both sides — if both
dashboard and source agree a role is unassigned, that is a VERIFIED match,
not a discrepancy.

### LAW F — Audit what the dashboard shows, not what the source has that the dashboard doesn't
This audit answers one question: *is what the dashboard displays accurate?*
It does not answer a different question: *is the source data complete
relative to the dashboard?* Those are separate audits. When a data point
exists only in the source and the dashboard renders nothing for it (a blank
description, a missing risk, a team role never surfaced), do not render a
comparison row with an empty dashboard column next to a full source column
— that visually misrepresents "absent by design or by known gap" as a
dashboard defect. Drop the row entirely. This has been applied twice so far
(Team sub-tab, Risk Register tab — see their respective sections) and
should be applied by default to any new sub-tab/tab added to this audit,
unless the person requesting the audit specifically asks for source-side
completeness to be checked too (that would be a legitimate but different
request — ask rather than assume). The reverse direction — dashboard shows
something the source doesn't have — is a different, genuine finding
(the dashboard is showing unbacked data) and must stay in the output.

---

## Resolving R2/R3 Sheet Names (do this before marking Overall Status/Snapshot UNVERIFIABLE)

The aggregated R2/R3 reports may not expose a Sheet Name column (see Known
Structural Gaps) — but every row still carries a numeric `sheet_id` in its
row-level metadata, and each `sheet_id` can be resolved individually:

```python
# No `columns` parameter needed — omitting it returns sheet_name AND
# the full underlying data (including historical rows) in one call.
result = get_sheet_summary(sheet_id=<id_from_row_metadata>)
# result.sheet_name -> e.g. "AK-OTOF - Overall Status"
# result.rows -> the FULL history for that project's Overall Status sheet,
#                including old monthly entries, not just the current one
```

**Critical filter:** these per-project sheets accumulate a full history of
monthly updates. Only rows where `Include in this month's update? == "Yes"`
represent the current snapshot that the R3 aggregated report (and therefore
the dashboard) actually reflects. Filter to `Yes` rows before comparing —
otherwise you will compare the dashboard's current-month view against years
of historical entries and manufacture false discrepancies.

Spot-checked this way against three programs this session (CD19/CD3 TCE,
Bimagrumab, N3pG4 complement modulation — note the last one only resolved
after case-normalizing the project name, see Known Structural Gaps), the
dashboard's Overall Status data matched the resolved source **exactly,
milestone-for-milestone, verbatim** in all three cases. This means the
correct default assumption is that Overall Status is likely accurate, not
that it's unverifiable — [UNVERIFIABLE] should be treated as a temporary
label reflecting incomplete resolution this pass, not a finding about the
dashboard.

Full resolution requires one `get_sheet_summary` call per distinct
`sheet_id` — 34 for R3, 27 for R2 in the last confirmed count, no overlap
between the two sets. This is a bounded, mechanical task (not open-ended
research) and should be completed in a dedicated pass rather than attempted
piecemeal — budget ~61 tool calls plus the "Include=Yes" filtering and
project-name matching per sheet.

---

## Extraction Method — Pulling Dashboard Data Out of the HTML

The dashboard is a single self-contained HTML file with inline
`<script type="text/babel">` React source. Top-level data is declared as
`const NAME = [...]` or `const NAME = {...}`. Do NOT try to regex-capture
these with a naive `.*` — nested braces/brackets and embedded strings
(including escaped quotes and unicode) will break a greedy or naive regex.
Use balanced-bracket parsing instead:

```python
def extract_const(text, name, start_hint=0):
    idx = text.find(f'const {name}=', start_hint)
    if idx == -1:
        idx = text.find(f'const {name} =', start_hint)
    eq = text.index('=', idx)
    j = eq + 1
    while text[j] in ' \t\n': j += 1
    open_ch = text[j]                      # '[' or '{'
    close_ch = ']' if open_ch == '[' else '}'
    depth = 0; in_str = False; str_ch = ''; esc = False; k = j
    while True:
        c = text[k]
        if in_str:
            if esc: esc = False
            elif c == '\\': esc = True
            elif c == str_ch: in_str = False
        else:
            if c in '"\'': in_str = True; str_ch = c
            elif c == open_ch: depth += 1
            elif c == close_ch:
                depth -= 1
                if depth == 0: k += 1; break
        k += 1
    return text[j:k]   # feed to json.loads()
```

First locate all `const [A-Z_]+\s*=` occurrences with a simple regex to find
byte offsets, then call `extract_const` with each name and offset as a hint
(some names are substrings of others or repeat — always pass the located
offset, not just the bare name, to avoid grabbing the wrong occurrence).

### Confirmed data structures in this dashboard build (re-verify every session — variable names may change between builds)

| Constant | Type | Contents | Feeds dashboard tab |
|---|---|---|---|
| `PROGRAMS` | array | one object per program: `name, rollup, priShort, modality, phase, pm, nextMilestone, nextMilestoneDate, riskCounts, workstreams[], snapshot[], risks[], gateMilestones{}, team{groups[],totalRoles}, narrs[], needsAttention, lastUpdated{}` | Programs (main list + drawer) |
| `FLAT_RISKS` | array | one object per risk: `desc, mitigation, assessment, likelihood, impact, currentStatus, category, targetDate, yell, program` | Risk Register |
| `ALL_SCHED` | object | keyed by project name → array of `{task, start, end, phase, type, seq, level, parent_name, isHeader, mfgExcluded, ...}` | Mfg Roadmap |
| `STATIC_ANALYSIS` | string | hardcoded portfolio-stats paragraph, generated at some prior build time — **check its embedded date against `FEED_DATE` every session; it goes stale** | Risk Register (header text) |
| `FEED_DATE` | string | date the live PROGRAMS/FLAT_RISKS data was last refreshed | (metadata only) |
| `LENSES` | array | `{id, label}` pairs defining the actual top-level tabs | navigation (defines audit scope) |

**Always re-derive `LENSES` from the file rather than assuming 4 tabs.**
This build had exactly `Programs`, `Mfg Roadmap`, `Risk Register`,
`Ask the Portfolio` (id `chat`) — a future build could add or rename tabs.

---

## Program Drawer Sub-Tabs (within the Programs tab)

Per program, the drawer conceptually has these sub-tabs. Map them to
`PROGRAMS[]` fields as follows — **do not assume additional tabs exist
without confirming a matching field first (LAW B)**:

| Requested sub-tab | Dashboard field | Source to verify against | Verifiable this session? |
|---|---|---|---|
| Overall Status | `workstreams[]` (ws/status/comment) | R3 Overall Status | **Historically NO** — see Known Structural Gaps |
| Snapshot | `snapshot[]` (task/onTime/timeframe) | R2 Snapshot Tasks | **Historically NO** — same gap |
| Team | `team.groups[].roles[]` (role/person) | R5 Project Team | **YES** — direct Project Name field in R5 |
| Timeline | `gateMilestones{}` (code → {ms, task}) | R6 Schedule Detail, rows under a "Milestones (PRISM)" parent with task name == gate code | **YES** — exact task-name+date match works |
| Decision | *(does not exist as a field)* | n/a | **N/A** — see LAW B; use decision-keyword workstream entries as nearest analog only |
| Supply Chain Node | *(does not exist as a field)* | S1 Intake column "Supply Chain Node" (index 15) | **N/A on both sides** — see Known Structural Gaps |

---

## Comparison Logic Per Sub-Tab / Tab

### Team (Programs tab)
```python
src_set = {(cl(e['role']), cl(e['person'])) for e in r5_raw_for_project}
for t in dashboard_team_entries:
    key = (cl(t['role']), cl(t['person']))
    if key in src_set: status = 'VERIFIED'
    elif any role-only match with different person: status = 'DISCREPANCY: person differs'
    else: status = 'DISCREPANCY: role/person pair not found in source'
# ALSO report the inverse: source (role,person) pairs never dedup'd correctly
# also check for entries in R5 not present in the dashboard at all
```

### Team (Programs tab) — Person/Name Matching (see LAW E)
```python
PLACEHOLDER = {'', 'na', 'n/a', '<<update>>', 'tbd'}

def extract_names(person_field):
    """Extract plain names from 'Name <email>; Name2 <email2>, Name3/Name4'
    strings, tolerating a literal wrapping quote pair. Returns
    (set_of_names, is_unassigned)."""
    if not person_field:
        return set(), True
    s = cl(person_field)
    if len(s) >= 2 and s[0] == '"' and s[-1] == '"':
        s = s[1:-1].strip()
    if s.lower() in PLACEHOLDER:
        return set(), True
    parts = [p.strip() for p in re.split(r'[;,/]', s) if p.strip()]
    names = []
    for p in parts:
        m = re.match(r'^(.*?)\s*<[^>]*>\s*$', p)          # strip <email>
        name = m.group(1).strip() if m else p.strip()
        name = cl(name).strip('"').strip()
        if name and name.lower() not in PLACEHOLDER:
            names.append(name)
    return set(names), (len(names) == 0)

def cl_role(s):
    """Role-name key: same wrapping-quote strip as person fields."""
    s = cl(s)
    if len(s) >= 2 and s[0] == '"' and s[-1] == '"':
        s = s[1:-1].strip()
    return s.lower()

# Build source lookup keyed by normalized role name -> set of names ever assigned
src_names_by_role = defaultdict(set)
src_roles_seen = set()
for e in r5_raw_for_project:
    role_key = cl_role(e['role'])
    src_roles_seen.add(role_key)
    names, _ = extract_names(e['person'])
    src_names_by_role[role_key].update(names)

for t in dashboard_team_entries:
    role_key = cl_role(t['role'])
    dash_names, dash_unassigned = extract_names(t['person'])
    src_names = src_names_by_role.get(role_key, set())
    role_tracked = role_key in src_roles_seen
    src_unassigned = role_tracked and not src_names
    dash_l, src_l = {n.lower() for n in dash_names}, {n.lower() for n in src_names}

    if dash_unassigned and (src_unassigned or not role_tracked):
        status = 'VERIFIED'                          # both sides agree: unassigned
    elif dash_l and dash_l.issubset(src_l):
        status = 'VERIFIED'
    elif dash_unassigned and src_names:
        status = 'DISCREPANCY: source lists a name, dashboard shows unassigned'
    elif dash_l and src_unassigned:
        status = 'DISCREPANCY: dashboard shows a name, source has this role unassigned'
    elif dash_l and src_l and (dash_l & src_l):
        status = 'DISCREPANCY: partial match — source also lists additional name(s)'
    elif dash_l and not role_tracked:
        status = 'DISCREPANCY: role not found anywhere in source for this project'
    else:
        status = 'DISCREPANCY: dashboard and source list different names for this role'
```
**Scope decision — see LAW F:** only report on team entries the *dashboard*
actually displays. Do NOT add rows for source (role, person) pairs that
exist in R5 but have no dashboard counterpart. Including those rows was
tried and explicitly removed after review; don't reintroduce them without
being asked.

At last full run: 3,022 dashboard team entries checked across 131 programs →
**2,992 VERIFIED (99.0%)**, 30 genuine DISCREPANCY. Of those 30, 22 shared
one root cause (a single project, "Weight Management Extended Release -
Tirzepatide", has no matching entry anywhere in R5 by that name — source
only tracks it as "Tirzepatide/Mounjaro/Zepbound") — report clustered
root-cause discrepancies as one finding with an affected-row count, not as
N independent findings, or the log becomes noise.

### Timeline (Programs tab)
```python
r6_by_task = group R6 rows for this project by cl(task_name)
for code, gate in dashboard_gateMilestones.items():
    dash_date = fmt_epoch_ms(gate['ms'])
    matches = r6_by_task.get(cl(code), [])
    if not matches: status = 'DISCREPANCY: no R6 row with this exact gate-code task name'
    elif dash_date in {cl(m['start'])[:10] for m in matches}: status = 'VERIFIED'
    else: status = 'DISCREPANCY: date mismatch'
```
This works because gate milestones are literal rows in R6 under a
"Milestones (PRISM)" parent, named exactly by their gate code (FTD, FHD,
FED, CD, FRD, FS, FA, FL, etc.) — confirmed by spot-checking AK-OTOF this
session (all 7 gates matched exactly).

### Mfg Roadmap tab (project-by-project, full task list)
```python
lookup = defaultdict(list)  # normalized task text -> [start dates] from R6
for e in r6_raw_for_project: lookup[norm(e['task'])].append(cl(e['start'])[:10])

for t in dashboard_ALL_SCHED[project]:
    if t['isHeader'] and not lookup.get(norm(t['task'])):
        status = 'N/A — structural header row'          # LAW D
    elif norm(t['task']) in lookup and dash_start in lookup[norm(t['task'])]:
        status = 'VERIFIED'
    elif norm(t['task']) in lookup:
        status = 'DISCREPANCY: task name matched, start date differs'
    else:
        status = 'DISCREPANCY: no R6 row with this exact task name'
```
At last run: 8,294 tasks checked across 118 projects → 6,657 VERIFIED,
1,353 genuine DISCREPANCY, 284 correctly excluded as headers. Also check
for projects with live R6 data (`r6_full[project]` non-empty) but zero
entries in `ALL_SCHED` at all — these are coverage gaps, not per-task
mismatches, and should be listed separately.

### Risk Register tab (project-by-project, full risk list)
```python
d_risks = [r for r in FLAT_RISKS if cl(r['program']) == project]
s_risks = r4_raw_for_project
for i in range(max(len(d_risks), len(s_risks))):
    d, s = d_risks[i] if i<len(d_risks) else None, s_risks[i] if i<len(s_risks) else None
    if d and s and cl(d['desc']) == cl(s['risk']): status='VERIFIED'
    elif d and not s: status='DISCREPANCY: present in dashboard, not found in source'
    elif s and not d: status='DISCREPANCY: present in source, missing from dashboard'
    else: status='DISCREPANCY: text differs'
```
**Scope decision — see LAW F:** if a risk exists only in R4 and the
dashboard shows nothing for it (`d is None`, i.e. both `dash_desc` and
`dash_mit` are blank), **drop that row entirely** rather than rendering a
blank dashboard column next to a full source column. Filter these out after
building the comparison, before rendering:
```python
rk['rows'] = [r for r in rk['rows'] if r['dash_desc'] or r['dash_mit']]
rk['mismatch_count'] = sum(1 for r in rk['rows'] if 'DISCREPANCY' in r['status'])
```
This also means a project can drop out of the Risk Register tab entirely if
*all* of its risk data was source-only (last run: 42 → 33 projects, 70 rows
removed). Recompute the tab's project-filter dropdown and shown-count from
the filtered set, and update the Discrepancy Log Summary the same way —
don't leave stale entries referencing rows that were removed from display.
The inverse case (`d` present, `s` missing — dashboard shows a risk R4
doesn't have) is a genuine, different finding and should stay: it means the
dashboard is showing something with no source backing, which is worth
flagging, not hiding.
**Known false-positive source:** curly vs. straight (and double-escaped)
quotation marks in program names between `PROGRAMS[].name` and
`FLAT_RISKS[].program` / R5 project-name keys (e.g. `MAPT siRNA-BBB ("806")`
vs a curly-quoted variant, or a literal `\"807\"` double-escaped variant)
will silently break the project-name join and produce a spurious
100%-mismatch for that project — **this hit both Risk Register and Team
sub-tabs for the same three MAPT programs**, confirming it's a join-key
issue, not category-specific. Always normalize before matching:
```python
def norm_quotes(s):
    return s.replace('\u201c', '"').replace('\u201d', '"').replace('\\"', '"')
```
Build a normalized-key lookup as a fallback whenever an exact project-name
key lookup misses, before concluding a project has zero source data.

### Ask the Portfolio tab
No fixed data table — it's a live LLM chat grounded in the same
PROGRAMS/FLAT_RISKS/ALL_SCHED already audited elsewhere. Audit only the
static suggested-prompt list (e.g. a `PILLS` array) for referencing real,
existing programs. State explicitly that any discrepancy already logged for
a program's data will silently propagate into whatever this chat says about
that program — this is an inherited-risk note, not a new independent check.

### STATIC_ANALYSIS text (rendered inside Risk Register)
Treat every quantitative claim in this string as an auditable fact. Compare
against live `PROGRAMS`/`FLAT_RISKS` counts (total programs, priority-tier
breakdown, Red/Yellow program lists and counts, risk assessment breakdown,
`needsAttention` count). Also check the claim's **internal arithmetic**
(do the stated sub-counts sum to the stated total?) independent of any
comparison to live data — this session found the text didn't even sum to
itself (3+17+96=116 ≠ stated 123). Always note the staleness gap between
the date embedded in this string and the live `FEED_DATE`.

---

## Known Structural Gaps (confirmed this session — re-verify, don't assume still true or still false)

| Gap | Detail | Status as of last check |
|---|---|---|
| R3 Overall Status (aggregated report) has no Sheet Name column | Live columns: Primary, Comments, Status, Scope, Status/Next Milestone, Dosage Form, + several disease-specific picklists — no VID for Sheet Name | Confirmed on the aggregated report; **resolvable per-project** via `get_sheet_summary(sheet_id)` on each row's row-level `sheet_id` — see "Resolving R2/R3 Sheet Names". 3 of 34 R3 sheets resolved this way, all 3 matched dashboard exactly |
| R2 Snapshot Tasks (aggregated report) has no Sheet Name column | Live columns: Primary, Timeframe (date), Timeframe (text), ATCOT, Area, Area Description | Same as R3 — resolvable per-sheet, not attempted at full scale (27 sheets) yet |
| The `cmc-portfolio-dashboard` skill documents its own known coverage gaps | Its §0.8 "Known Data Coverage Gaps" section states R3/R2/R4's partial program coverage is a **Smartsheet data population issue, not a dashboard bug** (sub-sheets were never created for most programs) | **Always read the dashboard skill's own gaps section before flagging low coverage as a new finding** — re-discovering an already-documented, already-explained gap and presenting it as new is misleading, even if the underlying numbers are real |
| S1 Intake "Supply Chain Node" column is unpopulated portfolio-wide | All 143 rows return the literal string "Supply Chain Node" (the column header) as the value | Confirmed — field has never been filled in |
| Dashboard PROGRAMS[] has no `decision` or `supplyChainNode` field | Confirmed via `sorted(programs[0].keys())` — 30 keys, neither present | Confirmed structural, not a data gap |
| Dashboard PROGRAMS[] `narrs` field is present but 0% populated | `[p for p in programs if p.get('narrs')]` → empty across all programs | Confirmed — field exists, never used |
| Two S1 programs missing from R4/R5 name-matching due to name drift | `GLP1 NPA Effort 4` (R4/R5) vs. S1's `GLP-1 Effort 4`; `MAPT siRNA-BBB Effort 4 (\"807\")` — a double-escaped-quote variant coexists with the correctly-quoted version in R5, splitting one program's team records across two keys | Confirmed, human resolution needed |
| Three programs (VERVE-102, VERVE-201, VERVE-301) have substantial live R6 schedule data (1,688 rows combined) but do not exist anywhere in S1 Intake | Likely an acquired-portfolio (Verve Therapeutics) integration gap, not a dashboard bug | Confirmed — portfolio-scope issue, not dashboard-rendering issue |
| R5/R6 row parser must handle TWO Smartsheet row-serialization formats | Any row with a cross-sheet-linked cell serializes as a YAML block instead of tab-delimited; a parser that only handles the tab format silently drops these rows with no error | Fixed in `gc-ivs-integrity-report` skill v2 — reuse that parser, don't reimplement |
| R5 person-name fields are NOT directly comparable to dashboard person fields | Email suffixes, inconsistent delimiters (comma/semicolon/slash), literal wrapping quote characters | Fixed this session — see LAW E and the Person/Name Matching reference implementation. **This was the single largest source of false-positive discrepancies found in this skill's history** — 99% of team entries were actually correct once matched properly, versus a majority-mismatched appearance before the fix |
| R3/R2 per-project underlying sheets contain full historical logs, not just the current snapshot | Rows carry an "Include in this month's update?" Yes/No flag; only Yes rows match what the aggregated report (and dashboard) shows | Confirmed on AK-OTOF (31 total rows, subset marked Yes) — always filter before comparing |

---

## Build Sequence

```
1.  Receive the dashboard HTML upload. Locate it at /mnt/user-data/uploads/.
2.  If a cmc-portfolio-dashboard skill file is also provided, diff its
    documented Smartsheet IDs against the ones this skill uses — confirm
    exact match before proceeding, and read its §0.8 (or equivalent) known
    coverage-gaps section so you don't re-report an already-documented gap
    as new. If no skill file is provided, state the source IDs used and
    proceed.
3.  Extract PROGRAMS, FLAT_RISKS, ALL_SCHED, STATIC_ANALYSIS, LENSES via
    balanced-bracket parsing (see Extraction Method). Validate each with
    json.loads() before proceeding — do not trust a partial/truncated grab.
4.  Confirm LENSES matches the assumed 4-tab structure; adjust section count
    if not.
5.  Pull live Smartsheet source: S1 (get_sheet_summary, Project Name +
    Modality + Status + Priority + Next Milestone + Next Milestone Date +
    Supply Chain Node), R4/R5/R6 (get_report, page_size=10000, paginate R6
    across 2+ pages as needed).
6.  Parse R4/R5/R6 with the dual-format (tab + YAML) parser from
    gc-ivs-integrity-report v2. Reconcile parsed count == declared
    total_row_count before trusting any of it (LAW 5 from that skill).
7.  Build project-name reconciliation: dashboard PROGRAMS names vs. live S1
    names (case/quote/whitespace-normalized, using norm_quotes — see Risk
    Register section) — log every 1:1 mismatch by name, don't just count them.
8.  Attempt R2/R3 per-project sheet-name resolution (see "Resolving R2/R3
    Sheet Names") for as many distinct sheet_ids as budget allows — full
    resolution is 61 calls; even a partial pass materially improves
    Overall Status/Snapshot coverage versus leaving it all UNVERIFIABLE.
9.  Run the five comparison routines (Team, Timeline, Mfg Roadmap, Risk,
    and Overall Status/Snapshot for any sheet_ids resolved in step 8) per
    the Comparison Logic section above. Use the LAW E name-matching logic
    for every person-field comparison, not just Team.
10. Verify STATIC_ANALYSIS claims against live PROGRAMS/FLAT_RISKS; check
    internal arithmetic independent of live-data comparison.
11. Assemble the four-tab HTML: Programs (by project) / Mfg Roadmap (by
    project) / Risk Register (by project) / Other Tabs & Summary. Each of
    the first three tabs gets its own project filter dropdown + text search
    + collapse/expand controls — do not ship this without them; the
    program/task counts make an unfiltered document unusable. Team sub-tab
    shows only dashboard-displayed rows (see Scope decision under Team).
12. Compile the Discrepancy Log Summary from every non-VERIFIED entry
    logged during steps 9-10, tagged by (project, sub-tab, detail). Cluster
    discrepancies sharing one root cause into a single log entry with an
    affected-count, don't list them as independent findings.
13. Write to /mnt/user-data/outputs/, present_files.
```

---

## Past Failure Modes

| Failure | Root cause | Prevention |
|---|---|---|
| Regex-based const extraction truncates or over-captures | Naive `.*?` or greedy patterns break on nested brackets / embedded quotes | Always use the balanced-bracket parser in Extraction Method |
| False "0 risks in dashboard" for a real program | Program name uses different quote characters (straight vs. curly) between `PROGRAMS[].name` and `FLAT_RISKS[].program` | Normalize + fuzzy-check before reporting a project as fully missing; report the quote-mismatch itself as its own finding |
| Reporting header/grouping rows as schedule discrepancies | `isHeader` flag not checked before comparison | Always branch on `isHeader` first (LAW D) |
| Claiming Overall Status/Snapshot discrepancies that aren't real | Source (R3/R2) can't be grouped by project this session; any "comparison" would be against fabricated matching | Mark UNVERIFIABLE, never guess (LAW C) |
| Auditing a "Decision" or "Supply Chain Node" tab that was assumed to exist | Field never confirmed against the actual PROGRAMS[] keys before building the comparison | Always run `sorted(programs[0].keys())` (or equivalent) before assuming a field/tab exists (LAW B) |
| STATIC_ANALYSIS treated as always-current because it's "in the dashboard" | It's a build-time snapshot string, not a live computed value | Always diff it against live PROGRAMS/FLAT_RISKS counts and check its own internal arithmetic |
| Output file unusable due to size with no navigation | 131-291 project blocks with no filter/search/collapse controls | Always include per-tab filter dropdown + search input + collapse/expand (step 9) |
| Team sub-tab showed a huge false-positive discrepancy rate | Exact string comparison between dashboard's plain names and R5's "Name <email>" format, plus unhandled semicolon/slash delimiters and literal wrapping quotes | Fixed via LAW E — normalize before comparing, never compare raw field strings for person data |
| Re-flagged an already-known, already-explained coverage gap as a new finding | Didn't check the `cmc-portfolio-dashboard` skill's own §0.8 "Known Data Coverage Gaps" section before characterizing low R2/R3/R4 coverage as a dashboard defect | Always read the paired dashboard skill's documented gaps before framing a coverage number as new |
| Marked Overall Status wholesale [UNVERIFIABLE] when it was actually resolvable | Assumed the aggregated-report Sheet Name gap meant no path to verification existed; didn't try calling `get_sheet_summary` directly on row-level `sheet_id` values | Always attempt per-sheet_id resolution before defaulting to UNVERIFIABLE — it worked 3/3 times when tried |
| Compared a per-project source sheet's full history against the dashboard's current-month view | Didn't filter on "Include in this month's update? == Yes" before comparing | Always apply this filter when using the per-sheet_id resolution path |
| Included source-only team rows ("not present in dashboard") in the Team sub-tab output | Conflated "is the dashboard accurate" with "is the source complete" — two different questions | Scope Team sub-tab strictly to what the dashboard displays; removed per explicit request, keep it removed — generalized as LAW F |
| Risk Register showed blank dashboard columns next to full source columns for source-only risks | Same root cause as the Team issue above, found independently in a different tab | Apply LAW F consistently — drop the row, recompute per-project counts and dropdown options, update the Discrepancy Log to match |
