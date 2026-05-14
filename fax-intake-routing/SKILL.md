---
name: fax-intake-routing
description: Classify and structure an incoming healthcare fax into a routable work-queue ticket. Use when the user pastes OCR'd fax content, references a fax file (PDF, TXT, or OCR output), or asks to categorize/route an inbound document. Extracts patient identifiers, ordering provider + NPI, HCPCS/ICD-10 codes, attachments detected, urgency signal, and outputs an organized HTML report with copy buttons next to every value the user might paste into another system. Triggers on phrases like "route this fax", "classify this fax", "categorize fax", "intake this fax", "process this fax", "fax routing", "where does this fax go".
---

# Fax Intake Routing

Turns raw fax content (OCR'd text, PDF, or pasted excerpt) into an organized HTML report. The page leads with the target queue, then lists every extracted field with a copy button so an intake coordinator can paste values straight into their downstream system (EHR, ticketing tool, etc.).

## When to invoke

- The user pastes fax content into chat and asks to classify or route it
- The user references a `.txt`, `.pdf`, or image file containing fax/OCR output
- The user asks something like "where does this fax go?", "intake this for me", or "what's in this document?"

## HIPAA + testing notes

**This skill is for testing.** Use only synthetic samples. Do not paste real PHI into this environment unless your AWS BAA is signed and Claude Code is pointed at Amazon Bedrock with that BAA in place (see the GenHealth setup walkthrough).

The output is **always a draft**. A human intake coordinator must review before the ticket dispatches to a queue. Default low-confidence tickets to a human review queue.

In the chat reply (after generating the HTML), do NOT echo raw PHI — name + summary line only. The HTML file itself contains the extracted data; the chat reply tells the user the file is ready and what queue it routed to.

## Workflow

1. **Locate the content.** Ask the user where the fax content is — pasted in chat, a file path, or described inline. If a file path is given, read it (Read tool for text, appropriate tool for PDF/image).
2. **Classify + extract.** Map to one of the categories below. Pull patient presence flags, ordering provider details (name, NPI, practice), HCPCS, ICD-10, attachments, urgency.
3. **Score confidence.** Float 0–1. Anything below 0.7 gets a `.callout.warn` in the Flags section with a one-line "why I'm uncertain" note.
4. **Build the body HTML** using the structure spec below.
5. **Build the raw JSON** matching the schema below — pretty-printed (2-space indent).
6. **Read the template** at `~/.claude/skills/_shared/output-template.html`.
7. **Substitute placeholders:**
   - `{{TITLE}}` → e.g. `Fax Routing · CPAP New Order`
   - `{{KICKER}}` → `Fax Intake Routing` (this is the small mark in the dark header bar)
   - `{{TIMESTAMP}}` → ISO local datetime, e.g. `2026-05-13 14:22 EDT`
   - `{{BODY}}` → the body HTML you built in step 4
   - `{{RAW_JSON}}` → the JSON from step 5 (HTML-escape `<`, `>`, `&` if any appear in string values)
   - `{{DATA_NOTICE}}` → `Synthetic test data` if the input clearly contains "SYNTHETIC" or comes from `examples/`. Otherwise `PHI · handle per HIPAA policy`.
8. **Ensure** `./skill-output/` exists in the current working directory (`mkdir -p ./skill-output`).
9. **Write** the file to `./skill-output/fax-routing-YYYYMMDD-HHMMSS.html` (use real local timestamp).
10. **Open** the file: `open "<absolute path to file>"`.
11. **In chat**, report (under 60 words): the queue routed to, the file path (absolute), and any flags. No raw PHI.

## Categories

- `new_order` — first-time equipment or service order
- `refill` — replacement / resupply request
- `cmn` — Certificate of Medical Necessity update
- `denial` — payer denial (route to your AR / appeals queue; if appealing, invoke the `draft-response` skill in `payer · appeal` mode to draft the letter)
- `unknown` — confidence too low to categorize

## JSON shape (for the raw-data section)

Return JSON matching this exact shape. Use `null` for unknown fields rather than omitting keys.

```json
{
  "category": "new_order | refill | cmn | denial | unknown",
  "subtype": "string — e.g. CPAP, oxygen, wheelchair, walker",
  "patient": {
    "name_present": true,
    "dob_present": true,
    "mrn_present": false,
    "insurance_present": true
  },
  "ordering_provider": {
    "name": "string|null",
    "npi": "string|null",
    "practice": "string|null"
  },
  "hcpcs": ["string"],
  "icd10": ["string"],
  "attachments_detected": ["sleep_study", "face_to_face", "rx", "lmn", "demographics"],
  "urgency": "standard | stat | post_discharge | hospice",
  "route_to_queue": "new_orders_<subtype> | refills | cmn_updates | denials | review",
  "flags": ["missing_signature", "illegible", "expired_doc", "insurance_unverified"],
  "confidence": 0.0
}
```

## HTML body structure — single-screen dashboard

The body fits entirely on one screen — no scrolling required. Layout: a **title-block** at the top (kicker + h1 + hero-mini on the right), then **two rows of three cards** (`cards-3` grid). All extracted data lives in the six cards. **Every value an operations person would paste into another system gets a copy button.**

Use the components defined in the template CSS: `title-block`, `hero-mini`, `cards-3`, `card` (variants: `.blue`, `.mint`, `.warn`), `field`, `field-tight`, `chip`, `callout`, `confidence` / `confidence-bar`.

### The six standard cards

| Row | Card 1 | Card 2 | Card 3 |
|---|---|---|---|
| **1** | Category & Confidence | Patient | Provider |
| **2** | HCPCS & ICD-10 | Attachments detected | Flags |

Concrete pattern (foot-orthotics new-order example — adapt to actual extracted values):

```html
<div class="title-block">
  <div>
    <span class="kicker">Fax routing result</span>
    <h1 class="title">New foot-orthotic order · routes to <em>new_orders_foot_orthosis</em></h1>
  </div>
  <div class="hero-mini">
    <div class="label">Target queue</div>
    <div class="value">new_orders_foot_orthosis</div>
    <div class="sub">3-page packet · BCBS PPO · standard urgency</div>
  </div>
</div>

<div class="cards-3">
  <div class="card blue">
    <h2>Category &amp; Confidence</h2>
    <div class="field field-tight">
      <span class="label">Category</span>
      <span class="value"><span class="chip blue">new_order</span></span>
      <button class="copy-btn" data-copy="new_order">Copy</button>
    </div>
    <div class="field field-tight">
      <span class="label">Subtype</span>
      <span class="value"><span class="chip mint">Foot Orthosis</span></span>
      <button class="copy-btn" data-copy="Foot Orthosis">Copy</button>
    </div>
    <div class="field field-tight">
      <span class="label">Urgency</span>
      <span class="value">standard</span>
      <span class="placeholder"></span>
    </div>
    <div class="field field-tight">
      <span class="label">Confidence</span>
      <span class="value">
        <span class="confidence">0.88
          <span class="confidence-bar"><span class="fill" style="width:88%"></span></span>
        </span>
      </span>
      <span class="placeholder"></span>
    </div>
  </div>

  <div class="card">
    <h2>Patient</h2>
    <div class="field field-tight">
      <span class="label">Name</span>
      <span class="value">Marie Curie</span>
      <button class="copy-btn" data-copy="Marie Curie">Copy</button>
    </div>
    <div class="field field-tight">
      <span class="label">DOB</span>
      <span class="value mono">1900-12-05</span>
      <button class="copy-btn" data-copy="1900-12-05">Copy</button>
    </div>
    <div class="field field-tight">
      <span class="label">Address</span>
      <span class="value">218 Forest Hills Ave, Boston MA 22180</span>
      <button class="copy-btn" data-copy="218 Forest Hills Ave, Boston MA 22180">Copy</button>
    </div>
    <div class="field field-tight">
      <span class="label">Insurance</span>
      <span class="value">BCBS PPO · <span class="mono">XYZ123456789</span></span>
      <button class="copy-btn" data-copy="XYZ123456789">Copy</button>
    </div>
  </div>

  <div class="card">
    <h2>Provider</h2>
    <div class="field field-tight">
      <span class="label">Prescriber</span>
      <span class="value">Arjun Raj, DPM</span>
      <button class="copy-btn" data-copy="Arjun Raj, DPM">Copy</button>
    </div>
    <div class="field field-tight">
      <span class="label">NPI</span>
      <span class="value mono">9182734556</span>
      <button class="copy-btn" data-copy="9182734556">Copy</button>
    </div>
    <div class="field field-tight">
      <span class="label">Practice</span>
      <span class="value">Boston Orthotics &amp; Prosthetics</span>
      <button class="copy-btn" data-copy="Boston Orthotics &amp; Prosthetics">Copy</button>
    </div>
    <div class="field field-tight">
      <span class="label">Fax</span>
      <span class="value mono">912-219-2311</span>
      <button class="copy-btn" data-copy="912-219-2311">Copy</button>
    </div>
  </div>
</div>

<div class="cards-3">
  <div class="card mint">
    <div class="card-h-row">
      <h2>HCPCS &amp; ICD-10</h2>
      <button class="copy-btn" data-copy-target="all-codes-payload">Copy all</button>
    </div>
    <!-- Hidden payload that the "Copy all" button copies as a single multi-line string.
         Format: "HCPCS: <list>\nICD-10: <list>" — friendly to paste into spreadsheets,
         tickets, or email bodies. -->
    <div id="all-codes-payload" style="display:none;">HCPCS: L3020 (LT, RT)
ICD-10: M19.071, M21.6x1, M21.6x2, M79.671, M79.672</div>
    <div class="field field-tight">
      <span class="label">HCPCS</span>
      <span class="value mono">L3020 <span class="chip muted">LT</span><span class="chip muted">RT</span></span>
      <button class="copy-btn" data-copy="L3020">Copy</button>
    </div>
    <div class="field field-tight">
      <span class="label">ICD-10</span>
      <span class="value mono">M19.071</span>
      <button class="copy-btn" data-copy="M19.071">Copy</button>
    </div>
    <!-- Additional ICD rows as needed; keep them tight. -->
  </div>

  <div class="card">
    <h2>Attachments detected</h2>
    <div class="field field-tight">
      <span class="label">Page 1</span>
      <span class="value"><span class="chip mint">Standard Written Order</span></span>
      <span class="placeholder"></span>
    </div>
    <!-- One field-tight row per attachment type. -->
  </div>

  <div class="card warn">
    <h2>Flags · review needed</h2>
    <!-- One small callout.warn per issue. Keep each to one short sentence.
         If there are no issues, replace with one cream callout: "Clean fax. All critical fields present." -->
    <div class="callout warn">
      <div class="label">Invalid date</div>
      <span>Prescriber signature date is 2/30/24 — not a real calendar date.</span>
    </div>
  </div>
</div>
```

## Rules of thumb

- **Single screen**: the six cards in a `cards-3` × 2 grid should fit a 1440×900 viewport comfortably. Use `field-tight` rows inside cards. If a card overflows, prune to the most decision-relevant 4–5 rows; supplementary detail belongs in the raw JSON section.
- **Copy buttons** on: NPI, HCPCS, ICD-10, member ID, address, prescriber name, practice — anything a coordinator would re-type. Skip copy buttons on display-only chips (category, urgency).
- **Confidence bar color**: mint default; add `.low` to the fill if `confidence < 0.7`; add `.med` if `0.7 ≤ confidence < 0.85`.
- **Flags card** uses the `.card.warn` variant (red top border) when any flag is set. Each flag becomes a small `.callout.warn` with a short specific issue. If no flags, use `.card` (default) and a single cream `.callout` saying "Clean fax. All critical fields present."
- **Provider card** should include the prescribing physician with NPI; the referring physician (if different) goes as a secondary row. Phone/fax: include at least one number with a copy button.
- **Copy-all on the codes card**: Use the `.card-h-row` header pattern with a `Copy all` button that targets a hidden `<div>` containing one short formatted block per code type (`HCPCS: ... \nICD-10: ...`). The pattern works for any other section that benefits from bulk copy (e.g., a full address block, an entire patient summary line). Single-field copy buttons still appear on each individual row.

## Reference example

`examples/synthetic-fax.txt` is the input. Run the skill on it and you should produce a CPAP new-order report routing to `new_orders_cpap`. The structure spec above describes that exact case.
