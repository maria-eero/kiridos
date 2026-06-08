# Dev Completed Summary

## Rules
1. Include the COMPLETE CSS from "CSS" section in every HTML file. Never skip or abbreviate styling.
2. Write the HTML file directly using the file-writing tool. NEVER generate Python/JS/shell scripts to build the HTML. If the content is too large for one write, split into multiple sequential writes (append mode) to the same file.
3. This file is the sole source of truth. Read fully before generating.
4. "CORE" and "ESN" are synonyms. Use "CORE" in display, "ESN" for Jira project key.

## Tools
- `jira_search`, `jira_get_issue` — ticket data
- `confluence_get_page`, `confluence_search`, `confluence_get_page_children` — RC cutoff dates

## Input
Ask user for **Release Version Name** (e.g., "26.0.1"). Normalize: if only two segments (e.g., "26.3"), append ".0" → "26.3.0". All other data is auto-discovered.
Dot releases (e.g., 26.0.1, 26.0.2) can be consolidated with the main version in one summary.

## Pipeline
Input(version) → 0a:VersionIDs → 0b:RCDates → 1:JiraFetch(2-pass) → 2:Merge+Compute → 3:Sort → 4:HTML(+filters) → 5:ValidateRC → 6:Save → 7:Check → 8:Upload(optional)

## Step 0a: Discover Jira Version IDs

**Find the release child page:** `confluence_search(query='title="eero mobile {VERSION}"', limit=5)` → match exact title, get page ID.

**Fallback:** If not found, browse children of the index page: `confluence_get_page_children(parent_id="5105614849")` for v26.x or `"5011308615"` for v6.x.

**Extract version IDs:** `confluence_get_page(page_id, convert_to_markdown=false, include_metadata=false)`. The page has two `<h2>` sections (android and iOS), each followed by a `<p>` with two links:
```
<a href="https://eeroinc.atlassian.net/projects/CORE/versions/{ID}/...">CORE tickets</a> |
<a href="https://eeroinc.atlassian.net/projects/SWS/versions/{ID}/...">SWS tickets</a>
```
Extract 4 IDs: CORE Android, SWS Android, CORE iOS, SWS iOS.

**Validate each ID:** `jira_search(jql="fixVersion={ID}", fields="fixVersions", limit=1)`. Verify name matches `"Android {version}"` or `"iOS {version}"`. If mismatch, STOP and alert user.

## Step 0b: Fetch RC Cutoff Dates

**Primary (both platforms):** `confluence_search(query='title="{platform} {version}"', limit=5)` → exact title match. Use full version string (e.g., "Android 26.3.0", "iOS 26.3.0").

**Fallback (if search returns empty):** Browse parent page children with pagination. Both parents have 90+ child pages — always paginate (start=0, 50, ...) until match found.
- iOS: `confluence_get_page_children(parent_id=29261893, limit=50)`
- Android: `confluence_get_page_children(parent_id=291340321, limit=50)` — exclude "[Ignore]" titles
- Android titles may include build numbers (e.g., "Android 6.25.0.36542")

**Extract:** `confluence_get_page(page_id, convert_to_markdown=false, include_metadata=false)` — must use HTML (not markdown) because iOS dates are in `<time datetime="YYYY-MM-DD">` tags that markdown drops. Parse RC table for LATEST RC (highest #, not RC 0). Android dates are in visible text.

Confirm with user: "Found {platform} {version} (Page ID: {id}). Latest RC #{num}, Cutoff: {date}. Correct?"

Store: Android RC#, Android date, iOS RC#, iOS date.

## Step 1: Fetch Jira Data (Two-Pass)

**CRITICAL:** Query each version ID separately — never combine in one query. Tickets can have multiple fixVersions, causing miscounts.

**Pass 1 — Metadata (4 calls):**
```
jira_search(jql="fixVersion={VERSION_ID} AND status='Dev Completed'",
  fields="key,summary,issuetype,priority,customfield_11787,parent", limit=50)
```
- `customfield_11787` = QA Assignee
- `parent` = Parent Epic

Confirm ticket counts with user.

**Pass 2 — PR data (N calls, one per ticket):**
```
jira_get_issue(issue_key="{KEY}", fields="customfield_10800")
```
Parse `customfield_10800.value` JSON for: `"state":"MERGED"|"DECLINED"`, `"lastUpdated":"YYYY-MM-DDTHH:MM:SS..."` (extract first 10 chars for date).

**BATCH LIMIT:** Execute Pass 2 calls in batches of no more than 10 parallel requests at a time. Wait for each batch to complete before starting the next. This prevents tool cancellation due to concurrency limits.

## Step 2: Merge & Compute

Per ticket:
- Key, Summary, Issue Type, Priority
- QA Assignee: `customfield_11787.value.displayName` or "Unassigned"
- Parent Epic: `parent.key` + `parent.fields.summary` or "-"
- PR Status: "MERGED YYYY-MM-DD", "OPEN", "DECLINED", or "NO PR"
- RC Availability — compare merge_date (YYYY-MM-DD string) to cutoff_date (YYYY-MM-DD string):
  - MERGED AND merge_date ≤ cutoff → YES
  - Everything else → NO
  - **Use YYYY-MM-DD string comparison only. Never compare full timestamps.**

## Step 3: Sort & Group

**CRITICAL:** Generate exactly 4 separate `<h2>` sections with their own tables, one per group:
1. CORE Android
2. CORE iOS
3. SWS Android
4. SWS iOS

Never combine CORE and SWS tickets in the same table, even if they share the same platform.

Within each group: Bugs → Tasks, then P0 → P1 → P2 → P3.

## Step 4: Generate HTML

**Title:** "Dev Completed Report - {VERSION}"

**Global filter chips (placed between `<h1>` and first `<h2>`):**
```html
<div class="global-filters">
  <a href="#" class="chip active" data-filter="reset">All</a>
  <a href="#" class="chip" data-filter="unassigned">Unassigned QA</a>
  <a href="#" class="chip" data-filter="not-rc">Not on RC</a>
  <a href="#" class="chip" data-filter="bugs">Bugs only</a>
  <a href="#" class="chip" data-filter="tasks">Tasks only</a>
  <a href="#" class="chip" data-filter="p0">P0 only</a>
  <label class="epic-dropdown">Epic: <select class="global-epic"><option value="">All</option></select></label>
  <button class="dark-toggle" title="Toggle dark mode"></button>
</div>
```

**Per platform section header:** `<h2>` "{PROJECT} {PLATFORM} {VERSION} (RC #{NUM}, Cutoff: {DATE}) - {COUNT} tickets"

**Filter bar (one per section, placed between `<h2>` and `<table>`):**
```html
<div class="filter-bar" data-section="{SECTION_ID}">
  <label>RC: <select class="filter-rc"><option value="">All</option><option value="YES">YES</option><option value="NO">NO</option></select></label>
  <label>PR: <select class="filter-pr"><option value="">All</option><option value="MERGED">MERGED</option><option value="OPEN">OPEN</option><option value="NO PR">NO PR</option><option value="DECLINED">DECLINED</option></select></label>
  <label>QA: <select class="filter-qa"><option value="">All</option><option value="unassigned">Unassigned only</option></select></label>
  <span class="filter-priority">Priority: <label><input type="checkbox" value="P0" checked> P0</label> <label><input type="checkbox" value="P1" checked> P1</label> <label><input type="checkbox" value="P2" checked> P2</label> <label><input type="checkbox" value="P3" checked> P3</label></span>
  <button class="filter-reset">Reset</button>
  <span class="filter-count"></span>
</div>
```
- `{SECTION_ID}`: unique id per section, e.g. `core-android`, `sws-ios`
- Each `<table>` must have a matching `data-section` attribute

**Table (8 columns, fixed widths 110/300/250/60/60/140/120/100px):**
Key | Summary | Parent Epic | Type | Priority | QA Assignee | PR Status | Available on RC{NUM}?

- Key: link to `https://eeroinc.atlassian.net/browse/{KEY}`
- Parent Epic: link epic summary to browse URL, or "-"
- All links: `target="_blank"`
- Priority: `<span class="priority p{N}">P{N}</span>`
- Last column header uses platform-specific RC number
- All `<th>` elements are clickable for sorting (add `class="sortable"`)
- Each `<table>` gets attribute `data-section="{SECTION_ID}"` matching its filter bar

**Appendix (before footer):**
```html
<div class="appendix">
  <h2>Appendix: Data Sources</h2>
  <p><strong>Release Pages (Confluence):</strong></p>
  <ul><li><a target="_blank" href="{ANDROID_CONFLUENCE_URL}">Android {VERSION}</a> | <a target="_blank" href="{IOS_CONFLUENCE_URL}">iOS {VERSION}</a></li></ul>
  <p><strong>Tickets Boards (Jira):</strong></p>
  <ul>
    <li><strong>CORE:</strong> <a target="_blank" href="{ESN_ANDROID_URL}">Android</a> | <a target="_blank" href="{ESN_IOS_URL}">iOS</a></li>
    <li><strong>SWS:</strong> <a target="_blank" href="{SWS_ANDROID_URL}">Android</a> | <a target="_blank" href="{SWS_IOS_URL}">iOS</a></li>
  </ul>
  <p style="color:#DE350B;font-weight:bold;margin-top:15px;padding-top:15px;border-top:2px solid #DE350B;">⚠️ This is a beta tool. Please double-check the project links above to ensure all Dev Completed tickets were filtered correctly and appear here.</p>
</div>
```

**Footer:** `<div class="footer">RC Availability Logic: A ticket is marked 'YES' if its PR was merged on or before the RC cutoff date. Tickets with DECLINED PRs or no PR data are marked 'NO'.</div>`

## Step 5: Validate RC Availability

For each MERGED PR, document: `{KEY}: {MERGE_DATE} <= {CUTOFF}? → YES/NO`. DECLINED/no PR → always NO. Show summary. Fix HTML if discrepancies found.

## Step 6: Save & Open

Write HTML file, then `open /path/to/file.html`.

## Step 7: Pre-Delivery Check (CRITICAL — re-read before generating final HTML)

Verify the generated HTML contains:
- 8 columns: Key(110px) | Summary(300px) | Epic(250px) | Type(60px) | Priority(60px) | QA(140px) | PR(120px) | RC(100px)
- Header format: "{PROJECT} {PLATFORM} {VERSION} (RC #{NUM}, Cutoff: {DATE}) - {COUNT} tickets"
- Exactly 4 sections: CORE Android, CORE iOS, SWS Android, SWS iOS
- Global filter chips div between `<h1>` and first `<h2>` with dark mode toggle
- Each section has: `<h2>` → `<div class="filter-bar">` → `<table>` with matching `data-section`
- Filter bar has: RC dropdown, PR dropdown, QA dropdown, Priority checkboxes, Reset button, count span
- All `<th>` have `class="sortable"` and sort arrows work on click
- Priority: `<span class="priority p{N}">P{N}</span>` with classes p0/p1/p2/p3
- QA "Unassigned" uses class `.unassigned`
- PR cells: `.pr-merged` / `.pr-open` / `.pr-declined`
- RC cells: `.rc-yes` / `.rc-no`
- All `<a>` tags have `target="_blank"`
- Sort: Bugs before Tasks, then P0→P1→P2→P3
- Appendix div and footer div present
- RC YES/NO matches Step 2 merge_date ≤ cutoff logic
- Complete CSS from "CSS" section included in `<style>` tag
- Complete JS from "JavaScript" section included in `<script>` tag before `</body>`

## Step 8: Confluence Upload (Optional)

Ask: "Summary complete! Do you need any changes? Would you like to add it to the Confluence page?"

If yes, upload to page ID `5309202581`:
1. Extract `<body>` content (exclude tags)
2. Strip all `class="..."` attributes (Confluence ignores custom CSS)
3. `confluence_update_page(page_id="5309202581", title="Dev completed tickets summary", content=<cleaned>, content_format="storage")`
4. Do NOT add inline styles — Confluence strips them with no visual benefit

Confirm: "✅ Summary uploaded! View: https://eeroinc.atlassian.net/wiki/spaces/QA/pages/5309202581/Dev+completed+tickets+summary"

## Manual Fallback

If auto-discovery fails, ask user for:
1. iOS Release Notes page URL (parent: https://eeroinc.atlassian.net/wiki/spaces/MOB/pages/29261893/iOS+Release+Notes)
2. Android Release Notes page URL (parent: https://eeroinc.atlassian.net/wiki/spaces/MOB/pages/291340321/Android+Release+Notes)
3. Jira version links (4 total). Provide the user with these pre-filtered search links (replace {VERSION} with the normalized version number):
   - CORE: `https://eeroinc.atlassian.net/projects/CORE?contains={VERSION}&selectedItem=com.atlassian.jira.jira-projects-plugin%3Arelease-page`
   - SWS: `https://eeroinc.atlassian.net/projects/SWS?contains={VERSION}&selectedItem=com.atlassian.jira.jira-projects-plugin%3Arelease-page`
   
   Ask them to share the 4 version links or just the ID number after "versions/" in the URL (e.g., `29010` from `.../versions/29010/...`):
   - CORE Android, CORE iOS, SWS Android, SWS iOS

Extract page IDs and version IDs from URLs.

## CSS

**MANDATORY — include this complete block in every HTML file.**

```css
body{font-family:Arial,sans-serif;margin:20px}h1{background-color:#0052CC;color:#fff;padding:15px}h2{background-color:#0052CC;color:#fff;padding:10px;margin-top:30px}table{border-collapse:collapse;width:100%;margin-bottom:30px;table-layout:fixed}th,td{border:1px solid #ddd;padding:8px;text-align:left;overflow:hidden}th{background-color:#f2f2f2;font-weight:bold}tr:nth-child(even){background-color:#f9f9f9}a{color:#0052CC;text-decoration:none}a:hover{text-decoration:underline}th:nth-child(1),td:nth-child(1){width:110px}th:nth-child(2),td:nth-child(2){width:300px;text-overflow:ellipsis;white-space:nowrap}th:nth-child(3),td:nth-child(3){width:250px}th:nth-child(4),td:nth-child(4){width:60px}th:nth-child(5),td:nth-child(5){width:60px}th:nth-child(6),td:nth-child(6){width:140px}th:nth-child(7),td:nth-child(7){width:120px}th:nth-child(8),td:nth-child(8){width:100px}.priority{padding:2px 8px;border-radius:3px;font-size:12px;font-weight:600;display:inline-block}.p0{background-color:#FFEBE6;color:#DE350B}.p1{background-color:#FFEBE6;color:#FF5630}.p2{background-color:#FFF0B3;color:#FF8B00}.p3{background-color:#DEEBFF;color:#0065FF}.unassigned{font-weight:bold;color:#DE350B}.pr-merged{color:#00875A}.pr-open{color:#FF8B00}.pr-declined{color:#DE350B}.rc-yes{color:#006644;font-weight:bold}.rc-no{color:#DE350B;font-weight:bold}.appendix{margin-top:40px;padding:20px;background-color:#F4F5F7;border-radius:4px;font-size:13px}.appendix h2{background-color:transparent;color:#333;padding:0;margin-bottom:15px}.appendix ul{list-style:none;padding:0}.appendix li{margin:8px 0}.appendix a{color:#0052CC;text-decoration:none}.footer{margin-top:30px;padding:15px;border-top:2px solid #ddd;color:#6B778C;font-size:12px}.filter-bar{display:flex;align-items:center;gap:12px;padding:10px 12px;background:#f4f5f7;border:1px solid #ddd;border-bottom:none;flex-wrap:wrap;font-size:13px}.filter-bar label{display:flex;align-items:center;gap:4px;white-space:nowrap}.filter-bar select{padding:3px 6px;border:1px solid #ccc;border-radius:3px;font-size:12px;background:#fff}.filter-priority{display:flex;align-items:center;gap:6px}.filter-priority label{cursor:pointer;font-size:12px}.filter-reset{padding:3px 10px;border:1px solid #ccc;border-radius:3px;background:#fff;cursor:pointer;font-size:12px}.filter-reset:hover{background:#e2e2e2}.filter-count{margin-left:auto;font-size:12px;color:#6B778C}th.sortable{cursor:pointer;user-select:none;position:relative}th.sortable:hover{background-color:#e2e2e2}th.sortable::after{content:"";display:inline-block;margin-left:4px;vertical-align:middle;border:4px solid transparent}th.sort-asc::after{border-bottom-color:#333;border-top:0}th.sort-desc::after{border-top-color:#333;border-bottom:0}.global-filters{display:flex;align-items:center;gap:8px;padding:12px 0;flex-wrap:wrap}.chip{display:inline-block;padding:5px 14px;border:1px solid #0052CC;border-radius:16px;font-size:12px;color:#0052CC;text-decoration:none;cursor:pointer;transition:all .15s}.chip:hover{background:#DEEBFF}.chip.active{background:#0052CC;color:#fff}.dark-toggle{margin-left:auto;padding:4px 10px;border:1px solid #ccc;border-radius:16px;background:#fff;cursor:pointer;font-size:16px;line-height:1}body.dark{background:#16163a;color:#e0e0e0}body.dark h1{background-color:#1a3a6b}body.dark h2{background-color:#1a3a6b}body.dark tr:nth-child(odd){background-color:#1a1a2e}body.dark tr:nth-child(even){background-color:#222238}body.dark th{background-color:#222238;color:#e0e0e0}body.dark td{border-color:#333;color:#e0e0e0}body.dark table{border-color:#333}body.dark a{color:#6ea8fe}body.dark .filter-bar{background:#1e1e3a;border-color:#333}body.dark .filter-bar select{background:#2a2a4a;color:#e0e0e0;border-color:#555}body.dark .filter-reset{background:#2a2a4a;color:#e0e0e0;border-color:#555}body.dark .filter-reset:hover{background:#3a3a5a}body.dark .appendix{background-color:#1e1e3a;color:#e0e0e0}body.dark .appendix h2{color:#e0e0e0}body.dark .footer{border-top-color:#444;color:#999}body.dark .chip{border-color:#6ea8fe;color:#6ea8fe}body.dark .chip:hover{background:#1a3a6b}body.dark .chip.active{background:#1a3a6b;color:#fff;border-color:#1a3a6b}body.dark .dark-toggle{background:#2a2a4a;border-color:#555;color:#e0e0e0}body.dark th.sortable:hover{background-color:#2a2a4a}.epic-dropdown{display:flex;align-items:center;gap:4px;font-size:12px;white-space:nowrap}.epic-dropdown select{padding:3px 6px;border:1px solid #0052CC;border-radius:3px;font-size:12px;background:#fff;color:#0052CC;max-width:250px}body.dark .epic-dropdown select{background:#2a2a4a;color:#6ea8fe;border-color:#6ea8fe}
```

## JavaScript

**MANDATORY — include this complete block in a `<script>` tag before `</body>` in every HTML file.**

```js
document.addEventListener('DOMContentLoaded',function(){if(localStorage.getItem('darkMode')==='true')document.body.classList.add('dark');window._gf={type:'',epic:''};function triggerAllFilters(){document.querySelectorAll('.filter-bar .filter-rc').forEach(function(sel){sel.dispatchEvent(new Event('change'))})}var darkBtn=document.querySelector('.dark-toggle');if(darkBtn){darkBtn.textContent=document.body.classList.contains('dark')?'☀️':'🌙';darkBtn.addEventListener('click',function(){document.body.classList.toggle('dark');var isDark=document.body.classList.contains('dark');localStorage.setItem('darkMode',isDark);darkBtn.textContent=isDark?'☀️':'🌙';triggerAllFilters()})}document.querySelectorAll('.filter-bar').forEach(function(bar){var sec=bar.getAttribute('data-section');var table=document.querySelector('table[data-section="'+sec+'"]');if(!table)return;var tbody=table.tBodies[0]||table;var allRows=Array.from(tbody.querySelectorAll('tr'));var rcSel=bar.querySelector('.filter-rc');var prSel=bar.querySelector('.filter-pr');var qaSel=bar.querySelector('.filter-qa');var priCbs=bar.querySelectorAll('.filter-priority input');var resetBtn=bar.querySelector('.filter-reset');var countSpan=bar.querySelector('.filter-count');var total=allRows.length;function applyFilters(){var rcVal=rcSel.value;var prVal=prSel.value;var qaVal=qaSel.value;var priVals=[];priCbs.forEach(function(cb){if(cb.checked)priVals.push(cb.value)});var shown=0;allRows.forEach(function(row){var cells=row.cells;var rc=cells[7]?cells[7].textContent.trim():'';var pr=cells[6]?cells[6].textContent.trim():'';var qa=cells[5]?cells[5].textContent.trim():'';var pri=cells[4]?cells[4].textContent.trim():'';var typ=cells[3]?cells[3].textContent.trim():'';var epc=cells[2]?cells[2].textContent.trim():'';var prMatch=!prVal||(prVal==='MERGED'?pr.indexOf('MERGED')===0:pr===prVal);var rcMatch=!rcVal||rc===rcVal;var qaMatch=!qaVal||(qaVal==='unassigned'&&qa==='Unassigned');var priMatch=priVals.length===0||priVals.indexOf(pri)!==-1;var typMatch=!window._gf.type||(window._gf.type==='Bug'?typ==='Bug':typ!=='Bug');var epcMatch=!window._gf.epic||epc===window._gf.epic;if(rcMatch&&prMatch&&qaMatch&&priMatch&&typMatch&&epcMatch){row.style.display='';shown++}else{row.style.display='none'}});countSpan.textContent=shown<total?'Showing '+shown+' of '+total:'All '+total+' tickets';restripe(tbody)}function restripe(tb){var vis=0;var dk=document.body.classList.contains('dark');Array.from(tb.querySelectorAll('tr')).forEach(function(r){if(r.style.display==='none')return;r.style.backgroundColor=vis%2===0?(dk?'#1a1a2e':'#fff'):(dk?'#222238':'#f9f9f9');vis++})}rcSel.addEventListener('change',applyFilters);prSel.addEventListener('change',applyFilters);qaSel.addEventListener('change',applyFilters);priCbs.forEach(function(cb){cb.addEventListener('change',applyFilters)});resetBtn.addEventListener('click',function(){rcSel.value='';prSel.value='';qaSel.value='';priCbs.forEach(function(cb){cb.checked=true});window._gf={type:'',epic:''};document.querySelectorAll('.chip').forEach(function(c){c.classList.remove('active')});var rc=document.querySelector('.chip[data-filter="reset"]');if(rc)rc.classList.add('active');var ge=document.querySelector('.global-epic');if(ge)ge.value='';applyFilters()});countSpan.textContent='All '+total+' tickets'});document.querySelectorAll('th.sortable').forEach(function(th){th.addEventListener('click',function(){var table=th.closest('table');var tbody=table.tBodies[0]||table;var idx=th.cellIndex;var rows=Array.from(tbody.querySelectorAll('tr'));var asc=!th.classList.contains('sort-asc');table.querySelectorAll('th').forEach(function(h){h.classList.remove('sort-asc','sort-desc')});th.classList.add(asc?'sort-asc':'sort-desc');var priOrder={'P0':0,'P1':1,'P2':2,'P3':3};rows.sort(function(a,b){var av=a.cells[idx]?a.cells[idx].textContent.trim():'';var bv=b.cells[idx]?b.cells[idx].textContent.trim():'';if(idx===4&&priOrder[av]!==undefined&&priOrder[bv]!==undefined)return asc?priOrder[av]-priOrder[bv]:priOrder[bv]-priOrder[av];if(idx===6){var ad=av.match(/\d{4}-\d{2}-\d{2}/);var bd=bv.match(/\d{4}-\d{2}-\d{2}/);if(ad&&bd)return asc?ad[0].localeCompare(bd[0]):bd[0].localeCompare(ad[0])}return asc?av.localeCompare(bv):bv.localeCompare(av)});rows.forEach(function(r){tbody.appendChild(r)});var sec=table.getAttribute('data-section');var bar=document.querySelector('.filter-bar[data-section="'+sec+'"]');if(bar){var tb=table.tBodies[0]||table;var vis=0;var dk=document.body.classList.contains('dark');Array.from(tb.querySelectorAll('tr')).forEach(function(r){if(r.style.display==='none')return;r.style.backgroundColor=vis%2===0?(dk?'#1a1a2e':'#fff'):(dk?'#222238':'#f9f9f9');vis++})}})});var epics={};document.querySelectorAll('tbody tr').forEach(function(row){var epc=row.cells[2]?row.cells[2].textContent.trim():'';if(epc&&epc!=='-')epics[epc]=true});var epicSel=document.querySelector('.global-epic');if(epicSel){Object.keys(epics).sort().forEach(function(name){var opt=document.createElement('option');opt.value=name;opt.textContent=name;epicSel.appendChild(opt)});epicSel.addEventListener('change',function(){window._gf.epic=epicSel.value;triggerAllFilters()})}document.querySelectorAll('.chip[data-filter]').forEach(function(chip){chip.addEventListener('click',function(e){e.preventDefault();var f=chip.getAttribute('data-filter');var bars=document.querySelectorAll('.filter-bar');bars.forEach(function(bar){bar.querySelector('.filter-rc').value='';bar.querySelector('.filter-pr').value='';bar.querySelector('.filter-qa').value='';bar.querySelectorAll('.filter-priority input').forEach(function(cb){cb.checked=true})});window._gf={type:'',epic:''};if(epicSel)epicSel.value='';document.querySelectorAll('.chip').forEach(function(c){c.classList.remove('active')});if(f==='reset'){document.querySelector('.chip[data-filter="reset"]').classList.add('active')}else{chip.classList.add('active');if(f==='p0'){bars.forEach(function(bar){bar.querySelectorAll('.filter-priority input').forEach(function(cb){cb.checked=cb.value==='P0'})})}else if(f==='not-rc'){bars.forEach(function(bar){bar.querySelector('.filter-rc').value='NO'})}else if(f==='unassigned'){bars.forEach(function(bar){bar.querySelector('.filter-qa').value='unassigned'})}else if(f==='bugs'){window._gf.type='Bug'}else if(f==='tasks'){window._gf.type='Task'}}triggerAllFilters()})})});
```
