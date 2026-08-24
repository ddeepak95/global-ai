---
name: sync-speaker-details
description: Sync Global AI Seminar speaker details from the Google Form responses sheet into the Speakers List tracker, then into the website's event-items.json and GitHub. Use when the user asks to "sync speaker details", "check for new speaker info/responses", "update the seminar website with new speaker details", or similar, for the Global AI Seminar series.
---

# Sync Global AI Seminar speaker details

Three-stage pipeline that moves a speaker's info from their Google Form
submission → the tracking spreadsheet → the public website JSON → GitHub.
Each stage is gated by a `Status` value in the tracker, so the skill is
idempotent: re-running it only processes rows that haven't reached the next
stage yet.

## Data sources (fixed paths — confirm they still exist; ask the user if not)

- **Form responses (source of truth for raw submissions)**: Google Sheet
  titled `Global AI Seminar Fall <year> Talk Details (Responses)`, in
  `G:\My Drive\Misc\Global AI\<academic year>\Events\`. This is a native
  Google Sheet (a `.gsheet` shortcut on disk, not a real file) — read it with
  the `mcp__claude_ai_Google_Drive__search_files` tool (query on the title,
  filter `mimeType = 'application/vnd.google-apps.spreadsheet'`) to get its
  `fileId`, then `mcp__claude_ai_Google_Drive__read_file_content` to get the
  full text of every response row. Do not truncate/summarize abstracts or
  bios when reading — copy them verbatim.
- **Uploaded photos from the form**: folder
  `G:\My Drive\Misc\Global AI\<academic year>\Events\Global AI Seminar Fall <year> Talk Details (File responses)\Photo (optional) (File responses)\`.
  Only present if a speaker attached a photo to their form submission.
- **Tracking spreadsheet ("Speakers List")**: a real `.xlsx` file (not a
  gsheet) at `G:\My Drive\Misc\Global AI\<academic year>\Events\Speakers List.xlsx`.
  Read/write it directly with Python + `openpyxl` (it's a normal file on
  disk under the Google Drive sync folder; saving it is enough, Drive syncs
  it). Columns (sheet `Sheet1`, header row 1):
  `Date | Name | Designation | Email | Status | Affiliation for Posters | Affliation | Photo | Talk Title | Speaker Bio | Talk Abstract | Relevant Link`
  - `Date` is like `Aug 25` (no year — infer the year from the semester
    folder name / context, e.g. "2026-27" academic year means Aug–Dec dates
    are in 2026 and Jan–May dates are in 2027).
  - `Status` drives the pipeline: empty/blank → `Details Received` →
    `Updated in Website`.
  - `Photo` holds a Google Drive share link to the speaker's photo (may be
    pre-filled by the organizer before the talk-details form is even sent).
  - Rows with no `Name`, or a `Name` like "No Seminar -- ...", are non-talk
    weeks — always skip them.
- **Website content**: `D:\Projects\PhD\global-ai\events\event-items.json`,
  a JSON array of event objects, one per seminar, git-tracked in
  `https://github.com/ddeepak95/global-ai`.
- **Website photos**: `D:\Projects\PhD\global-ai\events\photos\` (git repo
  copy, served via
  `https://raw.githubusercontent.com/ddeepak95/global-ai/main/events/photos/<filename>`)
  and its mirror `G:\My Drive\Misc\Global AI\<academic year>\Events\photos\`
  (kept in sync for the organizer's convenience — copy new photos to both).
  Filename convention: lowercase, hyphenated, first-name-last-name (e.g.
  `dipto-das.webp`, `aishwarya-agrawal.webp`), extension matches the source
  image. Keep whatever extension the source photo has.

## Stage A — Form response → Speakers List.xlsx (`Details Received`)

1. Read every row of the Responses gsheet (columns are typically
   `Timestamp | Title of the talk | Abstract | Speaker Bio | Photo (optional)`
   — there's no name field, so match each response to a speaker by
   timestamp order / process of elimination against xlsx rows that don't
   have a title yet, or ask the user to disambiguate if it's unclear).
2. For each response not yet reflected in the xlsx (i.e. the matching row's
   `Talk Title`/`Speaker Bio`/`Talk Abstract` are empty or differ):
   - Fill in `Talk Title`, `Speaker Bio`, `Talk Abstract` verbatim from the
     response.
   - If the response includes a photo upload, copy it from the File
     responses photo folder, rename to the `firstname-lastname.ext`
     convention, and place it in both photos folders (repo + Drive mirror).
     If `Photo` column is empty, fill it with a reference to the new file;
     if it already has a Drive link and no new photo was uploaded, leave it.
   - Set `Status` = `Details Received`.
3. Save the workbook in place with openpyxl (`wb.save(same_path)`).

## Stage B — Speakers List.xlsx → event-items.json → GitHub (`Updated in Website`)

Process every row whose `Status` is `Details Received` (this includes rows
Stage A just updated, and any left over from a previous run that didn't make
it to GitHub yet).

1. Match the xlsx row to its event object in `event-items.json` by date:
   convert the row's `Date` (+ inferred year) to `YYYY-MM-DD` and match
   against `start_date`. If no matching event object exists yet, create one
   at the correct position (chronological order) using the constant fields
   below.
2. Update the matched event object:
   - `name`: `"Global AI Seminar: " + Talk Title` (once a title exists; if
     for some reason there's no title yet, leave as plain `"Global AI
     Seminar"` and don't advance this row past Stage A).
   - `short_description`: `"**" + Full Name + "** is a " + Designation + " at " + Affiliation + "."` when only bio-teaser info is known (early placeholder state), OR the full `Speaker Bio` text from the xlsx once available — prefer the full bio text over the short teaser once it exists. **Always bold the speaker's name** (`**Name**`) at the point it first appears in the text, whichever form is used — this is a hard rule, don't skip it even when copying the bio verbatim from the form response.
   - `abstract`: `Talk Abstract` from the xlsx, verbatim.
   - `image_url`: `"https://raw.githubusercontent.com/ddeepak95/global-ai/main/events/photos/" + filename`, where `filename` matches whatever photo file exists for this speaker in `events/photos/`.
   - Constant fields (same for every seminar, don't ask — reuse existing
     entries as the reference): `event_url: ""`, `category: ["Seminar"]`,
     `time: "2:00 PM - 3:00 PM ET"`, `location: "Gates 203 | Virtual"`,
     `zoom_url` (copy from any existing entry — it's the standing Zoom
     link), `end_date` = `start_date`, `speakers: []`.
3. Only touch event objects that actually changed — don't rewrite the whole
   file's formatting/unrelated entries.
4. Validate the JSON parses after editing (`python -m json.tool` or
   equivalent) before committing.
5. `git status` first. Stage only `events/event-items.json` and any new
   files under `events/photos/`. Commit with a message describing which
   speaker(s) were added/updated. **Confirm with the user before `git push`**
   unless they've already told you in this conversation to push without
   asking.
6. Only after the push succeeds, go back to the xlsx and set that row's
   `Status` = `Updated in Website`, then save the workbook again.

## Notes

- Always re-open/re-read the xlsx right before writing to it — it's a
  shared file the organizer may also be editing by hand.
- If a row's data looks incomplete or ambiguous (e.g. can't tell which
  photo belongs to which speaker), stop and ask rather than guessing.
- Report a short summary at the end: which speakers moved to `Details
  Received`, which moved to `Updated in Website`, and which rows are still
  waiting on a form response.
