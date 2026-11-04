---
name: source-check
description: Grade, verify and add the sources cited in bpb's docs. Use when adding or changing a `[^label]` citation, when asked to audit sources or footnotes, or when asked to recheck dated claims (`recheck:` markers).
---

bpb's docs are its design, and a control is only as sound as the claim it rests on. This skill keeps every cited claim traceable to the best source available, and keeps dated claims from going stale silently. The footnotes in each page are the single source of truth; this skill keeps no other record.

## Reference

### Footnote format

A claim carries a `[^label]` marker. Its footnote sits in the block at the bottom of the same page, after the `---` rule. A label cited on several pages has an identical footnote on each.

```
[^label]: (grade) Author or org, "Title" (date): what this source supports. https://url
```

Reddit sources add `Read via Arctic Shift.` before the URL. Several URLs are joined with ` and `.

### Grades

The grade sits in parentheses at the start of the footnote. It rates the source, not the claim. Grade by what the page is:

- **primary**: the thing itself. A man page, spec, kernel or project documentation, source code, release notes, an option listing, a CVE record, the paper that reports the result.
- **reviewed**: a peer-reviewed or conference paper about someone else's system, or a court record.
- **secondary**: someone reporting on a primary source. Journalism, a summary site, a wiki, a review of a paper.
- **informal**: a forum thread, a blog post, a vendor's marketing, a personal write-up. Comments by a maintainer stay informal; they are expertise, not a record.

### Load-bearing

A claim is **load-bearing** when a control, a test in the roadmap, or a design decision rests on it. Changing or removing a load-bearing claim would change what bpb builds. Colour, history and examples are not load-bearing.

A load-bearing claim on a secondary or informal source is a **weak link**. The fix is a primary source that says the same thing; failing that, the text says the claim is unconfirmed.

### Recheck markers

A claim that will change with time (an open upstream PR, a version not yet in nixpkgs, a "verify on the pinned release") carries a marker right after its sentence:

```
<!-- recheck: 2027-03-01; closes when: nixpkgs #571181 is merged -->
```

The date is when to look next. The condition is observable: someone can check it and get yes or no.

### Fetching

Read every page before you rely on it. Use the `arctic-shift` skill for Reddit, and the web-fetch agent or `firecrawl` for everything else. A page that will not load is **unreachable**; say so, and leave its claims as they were. Fan many fetches out to subagents in batches, and have each return, per label, the sentence from the page that supports or contradicts the claim.

## Cite: adding a source

1. Read the source.
2. Write the claim so it says what the source says, and no more.
3. Add the marker and the footnote, with its grade, to the page. Reuse an existing label when the same source is already cited on another page, copying its footnote exactly.
4. Add a recheck marker if the claim will change with time.

Done when the footnote is graded, the URL loads, and the source supports the sentence as written.

## Audit: grading a page

Scope is one page, or one label across every page.

1. List every footnote in scope, and every sentence that cites each one.
2. Grade each footnote. Mark each citing sentence, not each footnote, load-bearing or not: one source can carry a control in one sentence and colour in another.
3. Verify every load-bearing citation against its page: **supports**, **partly** (with what is missing), **contradicts**, or **unreachable**.
4. For each weak link, search for a primary source that supports the claim. Record the replacement, or what you searched when none turned up.
5. Find claims that will change with time and have no recheck marker.
6. Report one table: label, grade, load-bearing, verdict, proposed action. Then list the proposed edits: grade tags, replacement sources, corrected wording, new recheck markers.
7. Apply the edits the user approves. Keep each footnote identical on every page that cites it.

Done when every footnote in scope has a row, every load-bearing citation has a verdict, and every weak link has a replacement or a recorded search.

## Recheck: dated claims

1. Find every `recheck:` marker in `docs/` whose date is today or earlier.
2. Check each condition against its source.
3. Closed: propose the new wording, and remove the marker. Open: propose the next date.
4. Report what changed, then apply the edits the user approves.

Done when every due marker is either closed or carries a future date.
