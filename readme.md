# Kaveri Clinic — Follow-up Register

**Problem, in two lines:** A small clinic writes follow-up dates on the patient's own paper card, so if the card is lost the follow-up is lost with it, and nobody can produce a list of who was due for review and didn't come. This register keeps that same information — visit, complaint, follow-up date — in one searchable place, and opens on the question staff actually ask each morning: *who needs a call today?*

Built for the SIH 2026 internal practical assessment (Loknath M, Reg 411625104035, PDKVCET, CSE PDKV).

---

## How to run it

No build step, no server, no install.

1. Download/clone the folder so `index.html`, `style.css`, `script.js`, and `appointments.json` sit **next to each other**.
2. Open `index.html` in a browser.
   - Double-clicking it works in most browsers.
   - If your browser blocks `fetch()` on a `file://` page, serve the folder instead (one line, no install needed):
     ```
     python3 -m http.server 8000
     ```
     then visit `http://localhost:8000`.
3. That's it — the register loads `appointments.json` and renders itself. No accounts, no database, no dependencies.

---

## What's on the screen

- **Who needs attention today** — a horizontal strip showing every patient who is *missed* or *due today*, sorted worst-overdue first, so the receptionist sees the people who need a phone call without scrolling through the full list.
- **Register** — the full list of 40 visits, with live search (filters as you type, no button) and four filter tabs: **All / Due Today / Upcoming / Missed**. A record count above the list always shows how many rows are currently visible, so a short list from a filter is never mistaken for missing data.
- **Detail view** — clicking any card opens a summary with the derived figure at the top, followed by every field for that visit.

---

## The data — `appointments.json`

40 realistic visit records. Every record has these fields:

| Field            | Meaning                                                                 | Values |
|------------------|--------------------------------------------------------------------------|--------|
| `appointment_id` | Unique ID for this visit row (`APT-20xx`).                              | string |
| `patient_id`     | Unique ID for the patient (`PID-01xx`). Lets the same person be found again across visits. | string |
| `patient_name`   | Patient's name.                                                          | string |
| `age`, `sex`     | Basic patient context, shown on every card and in the detail view.      | number, `M`/`F` |
| `contact`        | Phone number for calling the patient.                                   | string — **can be blank** if not yet collected |
| `visit_date`     | The date this visit already happened.                                    | `YYYY-MM-DD` |
| `complaint`      | What the patient came in for.                                            | string — **can be blank** if the patient never arrived |
| `attended`       | Whether the patient attended *this* visit.                               | `true` / `false` |
| `followup_date`  | The date the doctor advised the patient come back for review.           | `YYYY-MM-DD`, or **`null` if no follow-up was needed** (a one-time visit) |
| `notes`          | A short clinical note from the visit.                                   | string |

`age`, `sex`, and `notes` aren't in the assessment's required field list — they were added so the detail view has enough real context to be useful, the way an actual patient card would.

### The three deliberate awkward cases

The brief asks for these on purpose, because they're what force the interface to handle a value it can't compute cleanly, and what test whether search actually works. All three are labelled here so they're easy to find while testing:

1. **Missing value** — `APT-2010` (Devika Suresh) has `"contact": ""`. The detail view shows *"Not on file"* instead of a blank line, and the **Call patient** button is disabled rather than linking to nothing.
2. **Two very similar names** — `APT-2002` **Kavya Raman** and `APT-2027` **Kavya Ramanan** are two different patients (different `patient_id`, different visit). Searching `kavya` correctly returns both as separate, distinct records — proof the search matches on the real field, not on a rounded-off name.
3. **A record with nothing related to it** — `APT-2015` (Priyanka Rajagopal) has `"followup_date": null`: a one-time dressing change that needed no review. This is also what exercises Task 4's "record has no related entries" state — the detail view shows *"No follow-up scheduled"* at the top instead of a blank figure or an error.

There's also a fourth edge case worth knowing about: `APT-2008` (Nandini Rao) has `"attended": false` and `"complaint": ""` — a patient who never arrived, so there was nothing to record. The card carries a **No-show** tag in addition to its follow-up status, and the detail view shows *"Not recorded"* for the complaint.

---

## The derived figure: Days Overdue / Days Until Follow-up

Every record's status and headline number come from comparing its `followup_date` to today:

```
diff = followup_date − today   (in whole days)

diff < 0   →  "Missed"      →  shows "{-diff} days overdue"
diff = 0   →  "Due Today"   →  shows "Due today"
diff > 0   →  "Upcoming"    →  shows "{diff} days until follow-up"
followup_date is null → "No follow-up scheduled" (nothing to count — Task 4's no-related-entry case)
```

**"Today" is fixed at 25 July 2026**, not read from the system clock. That's deliberate: it means the Missed/Due/Upcoming split you see is exactly the same whether you run this now, during grading, or six months from now for a screenshot — it doesn't quietly drift as real time passes. To move the whole register to a different date, change one line: `TODAY` near the top of `script.js`.

### Checked by hand

Picked one record and worked it out on paper against `appointments.json`, to confirm the number staff would act on is actually right:

> **`APT-2001` — Meera Krishnan.** `followup_date` is `2026-07-09`. Today is `2026-07-25`.
> From 9 July to 25 July is 16 days. `diff = -16`, so the app should show **"16 days overdue"**.
> Opening that record in the app shows exactly **"16 days overdue"** — confirmed.

Two more spot-checks, for the other two states:
- `APT-2016` — Balamurugan Sethu, `followup_date = 2026-07-25` → same as today → **"Due today"**. Confirmed.
- `APT-2027` — Kavya Ramanan, `followup_date = 2026-08-02` → 8 days after today → **"8 days until follow-up"**. Confirmed.

---

## Screen states

| State | When it shows | What it says |
|---|---|---|
| Loading | While `appointments.json` is being fetched | A short animated loader, never a blank screen |
| Error | The file is missing, malformed, or the fetch fails | Plain explanation + a **Try again** button that re-runs the fetch |
| Empty (no matches) | A search or filter combination matches nothing | *"No matching records"* + a **Clear search & filters** button |
| No follow-up (per record) | A single record's `followup_date` is `null` | *"No follow-up scheduled"* in the same spot the derived figure would sit |

None of these ever leave a blank area or a spinner that doesn't resolve — every one of them was checked directly (see Testing, below).

## Accessibility & mobile

- Every status is shown as **text** ("Missed" / "Due today" / "Upcoming" / "No follow-up"), never colour alone — colour only reinforces what the label already says.
- Tap targets (buttons, filter tabs, the search field) are at least 44px tall on narrow screens.
- The layout reflows for a phone-width window with no horizontal scrolling; the only horizontal scroller is the "who needs attention" strip, which is intentionally a scrollable carousel, not a layout overflow.
- The detail view traps keyboard focus while open, restores focus to the card you opened it from on close, and the rest of the page is marked `inert` so screen-reader and keyboard users can't tab into content hidden behind it.
- A "Skip to register" link is the first focusable element, for keyboard users.
- Reduced-motion is respected (`prefers-reduced-motion`) — the small entrance/loader animations turn off.

## Testing

- Ran the full flow end to end against the real `appointments.json`: load → search → each filter tab → open a record → close it → error retry.
- Wrote a small headless test script (not part of the delivered app — just how this was checked) that loads the real page and data and asserts: the record count, the tally numbers, that both similar-name records open independently, that the missing-contact and no-follow-up records fall back correctly, and that a forced fetch failure produces the error state. All checks passed.
- Verified one derived figure by hand against the data (above).


- The register treats each visit row independently — if a patient's later visit already fulfilled an earlier row's follow-up, the earlier row will still show as missed, since nothing links the two rows together. A future version could group rows by `patient_id` and resolve a follow-up once a newer visit exists for the same patient.
- No way to mark a follow-up as "done" from the screen itself (this is a register/reference view, not a booking system) — updating `appointments.json` is currently the only way to change a record.