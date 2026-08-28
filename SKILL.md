---
name: linkedin-sales-nav-leads
description: "Extract leads from a live LinkedIn Sales Navigator people search into a spreadsheet when the user wants page-by-page collection from an open, signed-in browser session."
---

# LinkedIn Sales Navigator Leads

Use this skill when the user wants leads collected from a live LinkedIn Sales Navigator search, usually into an `.xlsx`, `.csv`, or checkpointed JSON file.

This skill is for browser-driven extraction from the user's existing Sales Navigator session. Do not use it for generic lead sourcing, web search, or cases where the user has not provided access to a live signed-in Sales Navigator search.

Do not confuse a request to collect business prospects with a request to recruit a salesperson or sales partner. Candidate sourcing may use the same live Sales Navigator surface, but it is a separate shortlist task: do not add candidates to a prospect workbook, save them to a list, or message them unless the user explicitly asks for those actions.

This skill also covers the common follow-up where the user wants a public LinkedIn profile URL added later. In that phase, the source of truth for the lead list is still the previously extracted sheet or checkpoint, but the public URL lookup should use a general web search result, not Sales Navigator profile URLs.

If the user first asks what you understood, asks you to hold off, or says something like `just let me know don't execute`, summarize the intended extraction plan and wait for explicit approval before collecting anything.

When the user asks for a demo, collect only the requested page range, save a demo workbook or checkpoint, then move the browser to the next page if requested. Do not continue into the full run until the user explicitly approves it.

Interpret shorthand in context. If the user says something like `Europe Recruiter's 1-10 headcount of all 1.5K+ leads`, treat `1-10` as the Sales Navigator company headcount filter and `all 1.5K+ leads` as a full paginated extraction request. Do not confuse that with "first 10 rows".

## Workflow

Claim and reuse the user's existing Sales Navigator search tab when it is already open. Prefer the in-app browser when that is where the search is active.

Before starting, record the live search name, current URL, visible page number, visible total-page label, and requested output name. Preserve the user's exact intended search filters; do not silently clear, broaden, replace, or save their search merely to make extraction easier.

For a full extraction, create and maintain a page ledger alongside the checkpoints. Each page must have one of these states: `complete`, `sparse`, `failed`, `skipped_by_user`, or `not_started`. A page is not complete merely because the browser navigated past it.

Work page by page:

- let each page finish loading before reading results
- collect the visible result cards on that page
- scroll the results container until the page has yielded all leads for that page
- checkpoint after every page so progress survives rate limits, browser resets, or tool timeouts
- move to the next page only after the current page has been saved

For long runs in browser-controlled environments, assume that "one go" usually means "one resumable chunk" rather than one literal uninterrupted tool call. The user may ask for "all pages at once," but the implementation should still be checkpointed and chunk-safe under the hood.

Do not make the user manually drive the run by repeatedly asking them to say `continue` or to name the next page. Once they have authorized the scope, advance autonomously through safe, bounded chunks, report progress at meaningful milestones, and resume from the ledger after an interruption. Explain any hard runtime or platform limit plainly, but keep working from saved state whenever the current turn permits it.

For long runs, prefer bounded, resumable chunks over one giant uninterrupted browser action. A good pattern is:

- process a small page range
- inspect what actually wrote to disk
- resume from the first missing or suspicious page

If the URL already contains a `page=` parameter, treat that as the browser's current page. For fresh full runs, start at page 1 unless the user asks to resume from the visible page or an existing checkpoint says otherwise.

When the user says `continue`, resume from the existing checkpoint rather than starting over. Preserve already-saved rows and pick up from the first unfinished page or first untouched row.

For full runs, prefer one checkpoint file per Sales Navigator page, for example `work/<run-name>/page_026.json`. This makes sparse or failed pages easy to repair without rewriting the whole run. After interruption, inspect the saved page files and resume from the first missing or invalid page.

When the user specifies an inclusive range such as pages `66` through `4`, write that range into the ledger before navigating. Visit every page in the requested direction, mark each one only after its checkpoint passes validation, and explicitly surface any page that could not be collected. Do not infer that a page was completed because an adjacent page succeeded.

When the user wants a sheet like the ones created in this task, default to these columns:

- `Name`: full visible lead name
- `Company`: company name shown on the result card
- `Position`: position/title as shown on LinkedIn

If the user asks for a specific sheet name, use it exactly for both the worksheet tab and the exported filename where practical.

For first-pass extraction, do not add a profile URL column unless it is directly available in a stable public form without guessing. Sales Navigator-only links do not qualify as public profile URLs.

When the user asks for multiple searches or regions in the same thread, keep outputs separate unless they explicitly request a combined workbook. Use filenames that reflect the user's requested names, normalized only as needed for the filesystem.

Recommended page checkpoint shape:

```json
{
  "saved_at": "ISO-8601 timestamp",
  "current_page": 12,
  "total_pages": 23,
  "page_label": "Page 12 of 23",
  "visible_card_count": 25,
  "clean_row_count": 25,
  "incomplete_count": 0,
  "rowCount": 25,
  "rows": [
    {
      "page": 12,
      "name": "Example Name",
      "company": "Example Company",
      "position": "Example Position"
    }
  ]
}
```

Recommended run checkpoint shape:

```json
{
  "saved_at": "ISO-8601 timestamp",
  "current_page": 28,
  "last_completed_page": 27,
  "total_pages": 93,
  "stopped_on_page": 34,
  "error": "playwright.evaluate exceeded its deadline"
}
```

For single-sheet jobs where the user mainly wants the final workbook, a plain JSON array of row objects is also acceptable if it is already in use. Reuse the existing shape instead of migrating formats mid-task unless there is a clear benefit.

## Extraction Guidance

Prefer reading the result cards from the live page rather than inventing URLs or scraping unrelated surfaces.

Use the company link on the card when available. Derive the position from the text line that contains the role and company, and keep it faithful to the displayed wording.

Deduplicate rows by a stable key such as:

`Name|Company|Position`

If the page uses a scrollable results pane, keep scrolling that pane until no new cards appear. Do not assume the initial visible cards are the full page.

Each Sales Navigator page is often a fixed batch such as 25 leads. Still verify the actual visible count on the page instead of assuming the expected number always loaded cleanly.

Sales Navigator lazily hydrates cards. A reliable structured pass is:

- scroll the results pane from top to bottom once to hydrate the page
- parse all result cards in one pass
- if the clean row count is unexpectedly `0` or far below the visible card count, wait, scroll again more slowly, and re-parse before saving

On badly hydrated pages, `visible_card_count` can already show the expected card count, for example `25`, while only a tiny subset of names or companies has actually hydrated. Treat "25 shells but 2 real rows" as a hydration failure, not a successful page.

Later pages in long runs can hydrate worse than early pages. When quality drops:

- reduce scroll jump size
- increase wait time between scrolls
- make a second slower pass
- prefer page-at-a-time repair over rerunning a large batch

Some pages virtualize card content while scrolling. In that mode, one scroll position may hydrate the top cards while another scroll position hydrates the bottom cards and dehydrates earlier ones. When that happens:

- collect rows incrementally across multiple scroll positions
- merge all complete rows seen during that pass
- deduplicate before saving the page checkpoint
- do not assume a single DOM read at the end of the scroll contains the full page

If the in-app browser's coordinate scroll is unreliable, prefer DOM-based scrolling of the page or results pane over blind screen-coordinate scroll attempts.

If direct DOM extraction becomes flaky or expensive on a heavily loaded page, use a lighter fallback instead of repeating failing page-script calls. A practical fallback is:

- use the visible page text as a snapshot source
- identify lead blocks from `Add <name> to selection`
- derive `Name`, `Company`, and `Position` from nearby lines
- merge those rows with any structured DOM rows
- deduplicate before saving

Treat that text-based parse as a resilience or repair technique, not the first choice when structured selectors are working normally.

Do not checkpoint a zero-row page as successful when the body still shows lead cards or the page footer still shows normal search results. Treat that as an under-hydration glitch and repair the page in place.

Do not treat “saved through the last page” as the same thing as “fully repaired.” A run can legitimately reach the last page while still leaving sparse pages that need another pass.

If LinkedIn shows `1K+` or another rounded total, do not treat it as an exact row count. Continue until the final accessible page or until LinkedIn stops returning new leads. Report the final saved count and any platform limit or throttle that prevented collecting more.

Use the visible pagination text such as `Page 1 of 67` when it is available. If `total_pages` cannot be read yet, do not treat that as the end of the run; keep the current page number from the URL or footer and retry the pagination read after the page settles.

If the footer becomes inconsistent, for example a bad label like `Page 93 of 92`, do not trust that footer alone. Prefer, in order:

- the current `page=` URL parameter
- the page checkpoint files that actually exist on disk
- the highest successfully saved page
- the footer text only as a hint

If a forward run keeps failing on one stubborn page, reverse pagination from a later stable page can be a valid recovery tactic. In that case:

- jump to the later page deliberately
- save pages in descending order
- still checkpoint each page independently
- resume from the first unsaved or suspicious page rather than assuming the direction change fixed everything

## Reliability

Keep an on-disk checkpoint after every page. In projectless work, prefer:

- `work/` for checkpoint JSON and helper scripts
- `outputs/` for the final user-facing workbook

If LinkedIn shows a throttle page such as `Too Many Requests`, stop collection, keep the checkpoint, and report:

- the current page
- the saved row count
- that the session can resume once the throttle clears

Do not silently restart from page 1 after a throttle unless the user asked for a fresh run or the earlier checkpoint is clearly unusable.

If the browser session resets, the browser tab changes, or the live page becomes unavailable, keep the checkpoint and reconnect to the existing tab or prompt the user only for the smallest next step needed.

When recovering browser state, remember that the active controlled Sales Navigator tab may still exist even if the user-tab listing is empty or a previous claimed-tab handle has gone stale. Check both:

- the browser's currently controlled tabs
- the user's open tabs

In long browser sessions, helper code or local runtime state can disappear even while the controlled Sales Navigator tab remains alive on the correct page. If that happens:

- rebind to the surviving controlled tab first
- restore only the minimal extraction helpers you need
- continue from the saved checkpoint frontier instead of restarting the run

Long Sales Navigator runs may hit browser-control or CDP timeouts even when LinkedIn itself is still usable. When that happens:

- check which page files were actually written
- check the live `page=` URL or pagination footer
- continue from the first missing or invalid page
- reduce the batch size, down to single-page steps if needed
- export a clearly named partial workbook if repeated timeouts prevent finishing the full run in the current turn

If the browser tool has a hard wall-clock timeout, expect a chunk to time out after writing several pages successfully. Treat the checkpoint directory as the source of truth, not the tool return value alone.

When a timed chunk ends on an error, inspect both:

- the last completed page recorded on disk
- the current live browser page

Those can legitimately differ by one or more pages if the tool timed out after navigation or after a save but before returning.

If clicking `Next` is flaky or ambiguous, use the current Sales Navigator search URL and set only the `page=` query parameter for the next page. Keep the rest of the search URL intact.

For very long searches, it is often better to:

- finish a first sweep across all accessible pages
- collect a repair queue of sparse pages
- repair only those sparse pages afterward

That usually works better than blocking the entire run on one stubborn page.

### Page Readiness and Repair

Before reading a page, wait for both the target `page=` URL (when present) and a non-skeleton search-results state. Re-read the pagination label after the page settles. If the URL, pagination label, and intended ledger page disagree, do not save a checkpoint until the mismatch is resolved or recorded as a failure.

Use this repair order for a sparse or blank page:

1. wait briefly and re-read the visible cards;
2. scroll the results pane in smaller increments to hydrate cards;
3. collect incrementally across scroll positions and merge complete rows;
4. reload or revisit only that page when the live URL can be verified;
5. put the page in the repair queue and continue with the remaining pages.

Do not repeatedly retry the same page in a tight loop. Two clean repair attempts are normally enough before deferring it, unless the user explicitly asks to keep trying.

When reverse pagination is used, preserve the same ledger and validation rules. A reverse sweep is a recovery strategy, not evidence that all pages above or below it were collected.

When enriching an already-exported workbook with public LinkedIn URLs, persist progress after every row rather than every page. Web search rate limits can interrupt the run much sooner than Sales Navigator page extraction.

Keep extraction and enrichment checkpoints separate. A Sales Navigator checkpoint is page-based; a public URL enrichment checkpoint should be query- or row-based and should survive browser tab resets.

## Public URL Enrichment

Use this section only when the user explicitly wants a public LinkedIn profile URL that opens without Sales Navigator.

Preferred method:

- keep the extracted lead list as the source rows
- search the web using a focused query built from `Name`, `Position`, and `Company`, for example `Name Position Company LinkedIn`
- inspect the search result page for a public `linkedin.com/in/` result
- save only confident matches

If the user says `do not use LinkedIn, use Google`, use Google search result pages only for lookup and do not open LinkedIn or Sales Navigator profile pages. It is still acceptable to save a public LinkedIn profile URL returned by Google.

When enriching multiple workbooks:

- read each workbook's actual headers instead of assuming one schema
- support common variants such as `Name`, `Full Name`, `Company`, `Position`, `Position / Title`, and `LinkedIn Profile URL`
- preserve valid existing public LinkedIn URLs and use them to seed the enrichment cache
- rename or add the requested URL column consistently, for example `LinkedIn` when the user asks for that exact column name
- write separate output workbooks unless the user asks for one combined file

Confidence rules:

- accept only public profile URLs on `linkedin.com/in/...` or a regional LinkedIn hostname with `/in/...`
- require the visible result text to match the lead name strongly
- prefer matches whose snippet or title also aligns with the company or role
- leave the URL blank if confidence is weak

Do not:

- copy Sales Navigator URLs
- guess profile URLs from name slugs
- force a match because the name is similar
- backfill uncertain rows with non-profile LinkedIn pages such as posts, people directories, company pages, or videos

When a public URL is added, use the user's requested column header. If they did not specify one, use a clear header such as `LinkedIn` or `LinkedIn Profile URL`.

Cache shape should distinguish three states:

- `found`: a confident public profile URL was saved
- `not_found`: the query was searched and no confident result was found
- `pending`: the query has not been searched yet

Do not treat a blank cached value as pending unless the existing checkpoint format gives no other choice. Otherwise the run will waste time re-searching rows that already had no confident match.

If Google or another search engine shows an anti-bot page, unusual traffic warning, CAPTCHA, or verification screen:

- stop the batch
- save all completed progress
- report the exact stopping point
- do not guess the remaining URLs

If browser policy requires user approval to solve a CAPTCHA or verification challenge, ask before doing so.

After the user clears a Google verification page and says `continue`, test one known query first. If normal results are available, resume from the enrichment checkpoint. If the controlled browser tab expired, open or claim a fresh Google tab and continue from the same cache.

## Quality Pass

Before exporting the final workbook, remove obviously malformed rows, such as placeholder parser output where:

- company is empty, `None`, or text like `in company`
- position is empty, `None`, or text like `in role`

Also review rows where either company or position is blank because of extraction glitches. If the missing value can be recovered directly from the visible lead card or live result page, patch it before final export.

Do not automatically drop every blank-company row. On this search surface, some legitimate leads can present a strong `Name` and `Position` while the company field is absent, self-branded, or visually inconsistent. Drop blank-company rows only when they are clearly malformed or unrecoverable parser junk.

Before final export, audit the page checkpoint directory for:

- missing page numbers
- page files with `rowCount: 0`
- duplicate rows across pages
- pages with a row count much lower than neighboring pages
- pages that finished far below the expected batch size such as `25` leads on this search surface

Repair suspicious pages from the live Sales Navigator page when possible. If repair is not possible, keep the partial workbook and report exactly which pages need another pass.

When a large number of pages are short but not empty, prefer an explicit repair pass over silently presenting the export as complete. The repair pass should focus only on short pages, not rerun already healthy pages.

When only a small repair queue remains, list those page numbers explicitly in the handoff.

If a page remains short after a clean repair pass and still looks stable on a second check, do not loop forever trying to force it to `25`. Treat that lower count as the current authoritative page result, keep the checkpoint, and call out the exact page numbers in the handoff.

Before handoff, compare:

- total rows across page checkpoint files
- deduplicated rows that actually make it into the workbook

These numbers can differ materially because cross-page duplicates are normal. Report both counts clearly rather than assuming they should match.

If the search itself includes regions the user did not mention, call that out in the handoff so the user knows the sheet reflects the live search filters.

For enriched workbooks, validate counts before handoff:

- total lead rows in each workbook
- rows with nonblank public LinkedIn URLs
- rows still blank because no confident result was found or lookup was interrupted
- cache count if enrichment can resume later

## Output Organization and Consolidation

Keep raw checkpoints, partial exports, and final deliverables distinct. Do not mix all three in one user-facing "final" folder.

When the user asks to combine completed searches, first identify which files are final versus partial/demo/source files. Then:

- merge only the intended final datasets;
- preserve `Source File` and inferred filters such as `Headcount` when available;
- deduplicate with a documented stable key while retaining provenance;
- name the merged workbook after the user’s requested category, such as `USA Recruiters.xlsx` for a combined US/USA recruiter set;
- place only the latest organized workbooks in a clearly named current folder and leave raw/source material in the existing archive location.

Never delete, overwrite, or move source files merely because a merged workbook was created. If replacing a prior final workbook, verify the replacement first and retain the original source collection as the archive.

For a current-deliverables folder, include only one workbook per intended category. Do not leave both a grouped workbook and its component workbooks there unless the user explicitly asks for both. If a requested geography or category has no collected workbook, say so; do not create a placeholder or imply coverage.

## Candidate-Sourcing Boundary

When the user asks to find a salesperson, sales partner, setter, or closer, treat it as a candidate-search task rather than a bulk lead-export run. Use the live results to build a small evidence-based shortlist, separating:

- full-cycle closers with demonstrated B2B/technology sales;
- appointment setters or cold callers who can create meetings but are not proven closers;
- adjacent consultants or agency owners who may be partnership or referral options rather than hires.

Do not claim that a candidate accepts a commission split, has access to leads, or will close the user’s offer unless the candidate has said so directly. Do not send messages, save leads, create a Sales Navigator list, or share the user's profile/contact details without explicit permission at the point of action.

## Deliverable

Export a clean workbook with a descriptive filename based on the search, sheet name, or region set. If the job is interrupted, keep both:

- the latest checkpoint in `work/`
- the latest workbook in `outputs/` when one already exists

For interrupted full runs, name the workbook honestly, for example `Europe Recruiter's 1-10 partial through page 26.xlsx`, and include the saved row count, highest completed page, and next resume page in the handoff. Do not describe a partial export as the full `1.5K+` result.

For completed first sweeps that still have sparse pages, be equally honest. A file like `US Recruiter Headcount 1 to 10 partial through page 93.xlsx` is acceptable when all pages were touched but a repair queue remains.

If the user explicitly says the collected set is "enough" and asks to finalize before the repair queue is done, export the workbook anyway, but state clearly that it is a collected-so-far or partial export and include:

- raw checkpoint row count
- deduplicated workbook row count
- the remaining sparse or suspicious pages
- the fact that the workbook does not represent the full accessible search yet

After writing the workbook, verify the actual output file on disk and verify the sheet row count from the saved file itself. Do not rely only on the export function returning success.

If helpful, also generate a small preview image or summary, but the spreadsheet is the primary deliverable.
