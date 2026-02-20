---
name: meeting-minutes
description: Transform raw meeting transcripts into professional structured minutes, then critique and deduplicate with user checkpoints at each stage. Flags risky merges for manual review using T1-T8 trigger system. Triggers - create minutes, meeting minutes from this, minutes from transcript, critique these minutes, clean up meeting notes.
---

# Meeting Minutes Skill

---

## WORKFLOW OVERVIEW

```
TRANSCRIPT INPUT
      │
      ▼
┌─────────────────────┐
│  PART 0: PREFLIGHT  │
│  Scan before draft  │
└─────────────────────┘
      │
      ▼
   PREFLIGHT CHECKPOINT
   Widget: confirm spellings,
   unknown speakers, open items
      │
      ▼
┌─────────────────────┐
│  PART A: CREATION   │
│  Draft minutes      │
└─────────────────────┘
      │
      ▼
   CHECKPOINT 1
   "Proceed to critique?"
      │
      ▼
┌─────────────────────┐
│  PART B: CRITIQUE   │
│  Flag duplicates    │
└─────────────────────┘
      │
      ▼
   CHECKPOINT 2
   Widget per flagged pair:
   Merge / Keep separate / Other
      │
      ▼
┌─────────────────────┐
│  PART C: FINALISE   │
│  Apply decisions    │
│  Remove flags       │
│  Output clean doc   │
└─────────────────────┘
```

---

# PART 0: PREFLIGHT

## Purpose

Scan the raw transcript before drafting anything. Surface problems that would corrupt the minutes if left unresolved — speakers, spelling variants, open items, off-topic content — then confirm with the user before proceeding. This catches issues at the cheapest possible moment.

---

## Preflight Checks

### 0.1 — Speaker Identification

Scan all speaker labels and list them as-is (initials, "Speaker 1", etc.) — do not attempt to map labels to real names. Infer roles only from context where possible.

**Mentioned but not present:** Scan the transcript for any person or party who is referenced but does not appear as a speaker. List them by role only (e.g. Structural Engineer, Contractor, Tile Manufacturer). Do not use names. If the role is unclear, flag it for the user to label at the preflight checkpoint.

Output format:
```
SPEAKERS IDENTIFIED:
  Speaker 1 — Lead Designer / Principal Designer
  Speaker 2 — Architectural Technologist
  Speaker 3 — UNKNOWN (appears 4 times, minimal content — likely observer)

MENTIONED BUT NOT IN ATTENDANCE (may receive actions):
  Structural Engineer
  Contractor
  Windows & doors supplier
  Tile manufacturer
  UNKNOWN ROLE — referenced at ~00:23:10, receives an action — user to label
```

Flag any mentioned party whose role cannot be determined. Use the widget at the preflight checkpoint to ask the user to provide a role label — not a name.

---

### 0.2 — Spelling Variants

Scan the entire transcript for names of people, companies, products, and technical terms that appear in more than one form. AI transcription errors, informal shorthand, and mishearing all create variants that must be unified before drafting — a single document should never use two different spellings for the same thing.

**Look especially for:**
- People whose name is spelled or transcribed differently across the transcript (e.g. "StructCo" / "Struct Co" / "Struck Co" / "Structure Co")
- People mentioned but not present, who may be named inconsistently (e.g. "BuildRight" / "the contractor" / "him")
- Product names used formally and informally (e.g. "TileCo SlimSlate Pro" / "the tile" / "SlimSlate")
- Abbreviations used inconsistently (e.g. "BCO" / "building control officer" / "building control")
- Technical terms with multiple phrasings (e.g. "beam and block" / "block and beam")
- Brand or company names garbled by transcription (e.g. "GlazeTech" / "GlazeTek" / "Glaze Tech")

For each variant group, propose a canonical form. Confirm with the user at the preflight checkpoint before drafting. Apply confirmed spellings consistently throughout — do not revert to variants.

Output format:
```
SPELLING / REFERENCE VARIANTS DETECTED:
  "StructCo" / "Struct Co" / "Struck Co" / "Structure Co"
    → Proposed canonical: "StructCo Ltd" — please confirm

  "TileCo SlimSlate Pro" / "SlimSlate" / "the tile"
    → Proposed canonical: "TileCo SlimSlate Pro"

  "BCO" / "building control officer" / "building control"
    → Proposed canonical: "BCO" — or prefer written out in full?

  "beam and block" / "block and beam"
    → Proposed canonical: "beam and block"
```

---

### 0.3 — Attribution Gaps

Identify any sections where it is genuinely unclear who is speaking and where the content is substantive enough to affect action ownership.

- Do not guess — flag the timestamp or approximate location
- Single-word filler responses with no action can be ignored
- If a decision or suggestion cannot be attributed, note it

Output format:
```
ATTRIBUTION GAPS:
  ~00:16:15 — "Okay." (Speaker 3) — single word, no action, will be ignored
  ~00:45:20 — Speaker 4 makes suggestion about battery storage — role unclear, may affect action ownership
```

---

### 0.4 — Off-Topic Content

Identify sections that contain no meeting content and will be excluded from the minutes:

- Social chat (greetings, personal conversations, family chat, birthdays)
- Technical difficulties (dropped calls, reconnecting, screen sharing problems)
- Background noise or interruptions with no substantive content
- Filler exchanges with no decisions or actions

User does not need to confirm these exclusions — just note them transparently so omissions are clear.

Output format:
```
OFF-TOPIC SECTIONS (excluded from minutes):
  00:00:00–00:00:24 — Call setup / opening pleasantries
  00:35:28–00:37:25 — Reconnection after dropped call / platform discussion
  01:21:49–01:22:09 — Birthday conversation
  01:51:22–01:52:43 — Closing pleasantries
```

---

### 0.5 — Open / Unresolved Items

Flag any topic that was raised but not resolved — no decision was made and no clear owner was assigned. These are candidates for a dedicated Outstanding Items section rather than being buried as "Noted."

Output format:
```
OPEN ITEMS DETECTED (no decision reached):
  - Roof cavity insulation thickness: 150mm vs 200mm — JW to reconsider
  - Battery storage location — subject to Client decision Wednesday
  - Drainage infiltration point — not yet resolved
```

---

## PREFLIGHT CHECKPOINT

Present the factual findings as a clear text summary, then use the **ask_user_input widget** for anything requiring a user decision.

Text summary first:

> "**Preflight complete.** Here's what I found:
>
> **Speakers confirmed:** [list with roles]
> **Mentioned but not in attendance:** [list — these may receive actions]
> **Off-topic sections excluded:** [list]
> **Open items (no decision reached):** [list or 'none — all items resolved']"

Then use the widget for decisions. Use one widget call with multiple questions where possible:

**For each unknown speaker or unidentified mentioned party**, include a single_select question — offer placeholder role labels, never request a real name:
> "I couldn't determine the role of [Speaker 3 / the party referenced at ~00:23:10]. How should I label them in the minutes?"
> Options: Observer / Client / Contractor / Consultant / I'll type a role label

**For each spelling variant group**, include a single_select question:
> "Canonical spelling for [name/term]?"
> Options: [variant A] / [variant B] / [variant C] / Other — I'll type it

Once all confirmations are received, apply canonical spellings and proceed to Part A.

---

# PART A: CREATION

## Purpose

Transform raw meeting notes or AI transcripts into professional structured minutes in numbered table format.

## Input Types

- AI-generated transcript (Teams, Otter, etc.)
- Handwritten notes
- Email summary of discussion
- Voice memo transcript

## Attribution

Apply confirmed name and role mappings from preflight throughout. If attribution remains unclear after preflight, use "Unattributed" rather than guessing.

- Designer vs Client usually clear from content (designer defends proposal, client requests changes)
- Technical queries vs commercial decisions indicate roles
- "We" typically means the speaker's organisation

## Output Format

### Header Table

| Field | Content |
|-------|---------|
| Project | [Project name/reference] |
| Date | [Date of meeting] |
| Attendees | [Names and roles — confirmed at preflight] |
| Mentioned / Not Present | [Names and roles — from preflight 0.1] |
| Meeting Type | [Teams/Site/Phone] - [Purpose] |
| Minutes By | [Who wrote them] |

### Item Tables (per section)

| No. | Item | Action |
|-----|------|--------|
| X.1 | [Decision/discussion point] | [Owner] |
| X.2 | [Next item] | [Owner] |

## Item Writing Rules

1. **One decision per item** — don't cram multiple things together
2. **Active voice** — "JW to provide..." not "It was agreed that fabrication would..."
3. **Specific** — include dimensions, materials, dates where mentioned
4. **Owner assigned** — every item needs someone responsible:
   - Use **"Noted"** for informational items with no action required
   - Use **"TBC"** for items where an action is needed but ownership is not yet agreed — do NOT bury these as "Noted"
   - Use **"Outstanding"** for open items flagged in preflight 0.5 — group these in a dedicated Outstanding Items section at the end of the minutes

## Section Grouping

Group items by topic/trade, not chronologically. Common sections:
- Drainage
- Structural
- Fire Safety
- Roof
- Walls & Insulation
- Landscaping
- Programme
- Commercial/Fees
- Communication

Add an **Outstanding Items** section at the end if any unresolved items were flagged in preflight 0.5.

---

## CHECKPOINT 1

After producing draft minutes, ASK:

> "Draft minutes complete. Should I now proceed to critique for duplicates and overlapping items?"

- If **yes** → proceed to Part B
- If **no** → output draft as final, end workflow

---

# PART B: CRITIQUE & DEDUPLICATION

## Purpose

Review draft minutes to identify duplicate, overlapping, or fragmented items. Flag risky merges for user decision rather than auto-merging.

## Duplicate Types

| Type | Description | Example |
|------|-------------|---------|
| **Semantic duplicate** | Same meaning, different words | "Remove wall section" vs "Remove masonry nib" |
| **Fragmented sequence** | Single action split across items | "Connect to column" / "Fill gap after removal" |
| **Overlapping scope** | Items partially cover same ground | "JW to handle drainage" vs "Drainage separate from retrofit" |
| **Cause-and-effect split** | Action and consequence separated | "Nib removed" / "Gap requires infill" |

## Detection Process

1. Work section by section
2. Identify items referencing the same physical element or decision
3. Map actions to elements
4. Check if actions are genuinely distinct or semantically equivalent

## Merge Confidence Check

### Trigger Conditions

| Code | Trigger | Risk |
|------|---------|------|
| T1 | Different action owners | Accountability muddled |
| T2 | Both items contain numbers/dimensions | Spec might be dropped |
| T3 | Conditional language (if/when/subject to/unless) | Condition lost |
| T4 | Different trades/disciplines | Context collapsed |
| T5 | >15 words length difference | Detail in longer item lost |
| T6 | Caveat/qualification present | Exception lost — **always flag even if solo** |
| T7 | Different timeframes | Timing confused |
| T8 | External document referenced | Link dropped |

### Confidence Threshold

| Triggers | Confidence | Action |
|----------|------------|--------|
| 0 | High | Auto-merge |
| 1 (not T6) | Medium | Auto-merge |
| **T6 alone** | **Low** | **Always flag — caveats hide liability** |
| **2+** | **Low** | **Flag for user decision** |

---

## CHECKPOINT 2

After identifying all flagged pairs, present them **one at a time** using the **ask_user_input widget**.

For each pair, show the full item text for both items in chat, then immediately follow with the widget:

> "Items X.X and X.Y are flagged as merge candidates [T1: different owners, T6: caveat present].
>
> **X.X:** [full item text]
> **X.Y:** [full item text]"

Widget — single_select:

**What should I do with these two items?**
- Merge them
- Keep separate
- Other — I'll tell you in chat

- If **Merge** → note for merge, preserve all unique detail, proceed to next pair
- If **Keep separate** → note to keep both, proceed to next pair
- If **Other** → pause and wait for user instruction before proceeding

Repeat for each flagged pair, then proceed to Part C.

---

# PART C: FINALISE

## Purpose

Apply all user decisions, clean up the document, output final .docx.

## Process

1. **For merged items:**
   - Combine into single item preserving ALL unique technical detail
   - Dimensions, materials, methods, constraints, relationships
   - Assign single owner (or note shared if genuinely split)
   - Renumber subsequent items

2. **For kept-separate items:**
   - Keep items exactly as-is

3. **Clean up:**
   - Renumber all items sequentially within sections
   - Verify no orphaned references
   - Confirm all canonical spellings from preflight are applied throughout
   - Confirm "Mentioned / Not Present" row is in the header table
   - Confirm Outstanding Items section is present if open items were flagged

4. **Output:**
   - Generate clean .docx
   - Present to user

## Merged Item Structure

```
[Element]: [Action] - [Key details]. [Rationale if relevant].    [Owner]
```

Preserve:
- Dimensions (150mm, 300mm, 5m)
- Materials (steel, concrete, timber)
- Methods (welded, concreted, bolted)
- Constraints (no new foundation, subject to survey)
- Relationships (connect to X AND Y)

Drop:
- Repetition in different words
- Implicit information (gap obvious if nib removed)
- Hedging language (preference to, proposal to)

---

# REFERENCE

## Common Merge Patterns

### Rejection-Then-Alternative
Before: X.1 "Option A rejected" / X.2 "Option B preferred"
After: X.1 "Option B selected. Option A rejected due to [reason]."

### Physical-Then-Consequence
Before: X.1 "Remove element" / X.2 "Fill gap"
After: X.1 "Remove element and make good."

### Who-Then-What
Before: X.1 "BT to handle drainage" / X.2 "Drainage includes [scope]"
After: X.1 "Drainage ([scope]): BT to handle."

### Caveat-Then-Content
Before: X.1 "Note required" / X.2 "Note to state [text]"
After: X.1 "Drawing note required: '[text]'"

---

## Quality Checklist

Before finalising:
- [ ] All canonical spellings from preflight applied throughout — no variants remain
- [ ] "Mentioned / Not Present" row included in header table
- [ ] All merged items preserve unique technical details
- [ ] No dimensions, materials, or methods lost in merges
- [ ] Actions have clear owners — Noted / TBC / Outstanding used correctly
- [ ] Outstanding Items section present if open items were flagged at preflight
- [ ] Rationale preserved where it aids understanding
- [ ] Items renumbered sequentially within all sections
- [ ] No orphaned cross-references
- [ ] Document formatted consistently

---

## QUICK WORKFLOW REFERENCE

```
0. PREFLIGHT
   0.1 Identify speakers + roles
       → List mentioned-but-not-present parties separately
   0.2 Detect spelling/name variants → propose canonical forms
   0.3 Flag attribution gaps
   0.4 Note off-topic sections (excluded silently)
   0.5 Flag open/unresolved items
   → PREFLIGHT CHECKPOINT (widget: confirm spellings, unknown speakers)

1. PART A: Draft minutes
   → Apply canonical spellings throughout
   → Use TBC / Outstanding for unowned/unresolved items
   → Add Outstanding Items section if needed

2. CHECKPOINT 1: "Proceed to critique?" (text prompt)

3. PART B: Critique — identify flagged pairs

4. CHECKPOINT 2: Per flagged pair — widget:
   → Merge / Keep separate / Other

5. PART C: Apply decisions, clean up, output .docx

6. END
```
