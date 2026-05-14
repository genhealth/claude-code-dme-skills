---
name: prior-auth-drafting
description: Review a prior-auth case against the payer's LCD or coverage policy BEFORE drafting — issue a verdict (SUBMIT / NEEDS DOCS / DON'T SUBMIT) based on whether the requested HCPCS is in scope and whether the clinical evidence meets each coverage criterion. If the verdict is SUBMIT or NEEDS DOCS, draft the medical-necessity narrative citing each met criterion. If DON'T SUBMIT, output the specific reason (wrong LCD, statutory non-coverage, missing required evidence) and a "what to do instead" recommendation. The payer policy can be pasted text OR a CMS LCD URL the skill fetches and extracts itself. Use when the user has a requested code + diagnosis + clinical findings + a payer policy (text or LCD link) and wants to know whether to submit. Triggers on phrases like "review this prior auth", "should we submit", "pa review", "draft prior auth", "draft medical necessity", "check coverage scope", "check this LCD", "pull the policy from this link", "PA narrative", "medical necessity letter", "write a PA for this patient", "is this case viable".
---

# Prior-Auth Review (with Conditional Drafting)

A pre-submission review skill. **Always runs the coverage check first.** Only drafts the narrative if the case is viable. If the case is not viable, the skill says so clearly and tells the user what to do instead — saving hours that would otherwise be spent drafting an appeal that's structurally doomed.

## When to invoke

- The user has a requested HCPCS code + ICD-10 + clinical findings + payer policy (pasted text or a CMS LCD link) and wants to know whether to submit
- The user asks "should we submit this PA?" / "review this case" / "is this viable?"
- The user drops in a CMS LCD URL and asks you to check the case against it
- The user pastes findings from fax-intake-routing that suggest a PA is needed and wants the next step
- The user explicitly asks for a medical-necessity narrative — still run the review first; the narrative is the second half

## HIPAA + testing notes

**Synthetic data only for testing.** Real PHI requires a signed BAA. The output contains patient identifiers and clinical detail — handle accordingly.

**Citations are traceable, not invented.** Only cite policy sections, study findings, and dates explicitly provided by the user. No invented LCD numbers, studies, or thresholds.

**The clinician signs any draft.** Every narrative is a draft awaiting clinician review.

## Required inputs

If any are missing, ask once with a bulleted list.

| Input | Example |
|---|---|
| Patient context | "55yo F, BMI 32, hx HTN" |
| Diagnosis (ICD-10) | "M19.071 OA right ankle/foot" |
| Requested HCPCS | "L3020" |
| Side / quantity / parameters | "bilateral, qty 1 ea" |
| Payer + policy | "Medicare · LCD L33686" — paste the relevant sections, **or** give the CMS LCD URL and the skill fetches it (see *Fetching a policy from a URL* below) |
| Clinical findings | each with source doc + date |
| Prescribing physician | name + NPI |
| Planned DOS | "2026-06-02" |

## Fetching a policy from a URL

If the user provides a **CMS LCD URL** — e.g. `https://www.cms.gov/medicare-coverage-database/view/lcd.aspx?LCDId=33686` — or another public payer-policy URL instead of pasting the text, fetch and extract it yourself:

1. **Download the page** with `curl` (CMS pages are slow for some fetch tools but reliably curl-able — use `curl`, not WebFetch):
   ```bash
   curl -s -L -A "Mozilla/5.0" "<policy URL>" -o /tmp/policy.html
   ```
2. **Strip scripts/styles/tags** and pull the sections that matter. For a CMS LCD:
   - **"Coverage Indications, Limitations, and/or Medical Necessity"** — the actual criteria
   - **The CPT/HCPCS code list** — to confirm whether the requested code is in scope
   - **The ICD-10 "supports / does not support medical necessity" lists**
3. **Confirm the extracted scope with the user before proceeding** — echo back the LCD title, the covered-HCPCS summary, and whether the requested code is in scope. Let the user sanity-check you grabbed the right policy and the right sections.
4. From there, treat the fetched text exactly like pasted policy text — same criteria mapping, same "no invented citations" rule. Cite the LCD ID and the section numbers you actually extracted.

## The verdict states

- **SUBMIT** (green) — code is in policy scope AND every required criterion has matching evidence. Draft the narrative.
- **NEEDS DOCS** (amber) — code is in scope, most criteria met, but a small number need additional documentation. Draft the narrative with placeholders and list what to obtain before submission.
- **DON'T SUBMIT** (red) — code is out of LCD scope, statutorily non-covered, or a hard criterion is unmet with no path to fix. Suppress the narrative; output explicit "what to do instead" guidance.

## Workflow

1. **Verify inputs**, especially the policy. If the user gave a CMS LCD URL (or another payer-policy URL) instead of pasted text, fetch and extract it first — see *Fetching a policy from a URL* above. If no policy is provided at all, ask for it.
2. **Coverage scope check** — does the requested HCPCS appear in the policy's covered-codes list?
   - If NO → verdict is **DON'T SUBMIT**. Skip criteria evaluation. Generate the "alternatives" guidance.
   - If YES → proceed.
3. **Criteria mapping** — for each policy criterion, find a matching clinical finding (source doc + date). Track unmet criteria as `missing_evidence`.
4. **Determine verdict**:
   - All criteria met → **SUBMIT**.
   - 1–2 criteria need additional doc → **NEEDS DOCS**.
   - A hard criterion is structurally unmet → **DON'T SUBMIT**.
5. **Build the body HTML** following the structure spec below.
6. **Build the raw JSON** matching the schema below.
7. **Read the dashboard template** at `~/.claude/skills/_shared/output-template.html`.
8. **Substitute placeholders:**
   - `{{TITLE}}` → e.g. `Prior-Auth Review · L3020 vs LCD L33686`
   - `{{KICKER}}` → `Prior-Auth Pre-Submission Review`
   - `{{TIMESTAMP}}` → ISO local datetime
   - `{{BODY}}` → the body HTML
   - `{{RAW_JSON}}` → JSON (pretty, 2-space)
   - `{{DATA_NOTICE}}` → `Synthetic test data` or `PHI · handle per HIPAA policy`
9. **Ensure** `./skill-output/` exists.
10. **Write** to `./skill-output/pa-review-<PatientLast>-YYYYMMDD-HHMMSS.html`.
11. **Open** the file.
12. **In chat**, report (under 60 words): verdict + 1-line reason + PDF path. If DON'T SUBMIT, also state the recommended alternative pathway.

## HTML body structure — single-screen dashboard

Layout: `title-block` (verdict prominently in `hero-mini`) → `cards-3` (Request / Coverage scope / What to do or Criteria) → `cards-3` (Criteria / Evidence in chart / Risks & flags). Optionally a final conditional `card.blue` containing the drafted narrative letter (only when verdict is SUBMIT or NEEDS DOCS).

The hero verdict color matches the state:
- SUBMIT → `hero-mini` with default styling, value chip mint
- NEEDS DOCS → `hero-mini` with amber border, value chip amber
- DON'T SUBMIT → `hero-mini` with `style="border-color: var(--warn); background: var(--warn-soft);"` and value text in `var(--warn)`

```html
<div class="title-block">
  <div>
    <span class="kicker">Prior-auth pre-submission review</span>
    <h1 class="title">Don't submit · <em>L3020 is out of LCD scope</em></h1>
  </div>
  <div class="hero-mini" style="border-color: var(--warn); background: var(--warn-soft);">
    <div class="label" style="color: var(--warn);">Verdict</div>
    <div class="value" style="color: var(--warn);">DON'T SUBMIT</div>
    <div class="sub">L3020 (foot orthosis) is outside LCD L33686, which covers ankle-foot &amp; knee-ankle-foot orthoses only.</div>
  </div>
</div>

<div class="cards-3">
  <div class="card">  <!-- Request summary, with Copy all (patient + code + ICD + prescriber + DOS) --> </div>
  <div class="card warn">  <!-- Coverage scope check: policy, in-scope HCPCS chips, requested chip in warn, "why it would be denied" callout --> </div>
  <div class="card blue">  <!-- What to do instead (DON'T SUBMIT only) OR Criteria summary (SUBMIT/NEEDS DOCS) --> </div>
</div>

<div class="cards-3">
  <div class="card">  <!-- Criteria checklist (PASS / FAIL / N/E per criterion) --> </div>
  <div class="card mint">  <!-- Evidence in chart (what's documented, with source doc + date) --> </div>
  <div class="card">  <!-- Risks & flags (code mismatch, statutory exclusion risk, doc quality, data irregularities) --> </div>
</div>

<!-- Conditional: only rendered when verdict is SUBMIT or NEEDS DOCS -->
<div class="card blue" style="margin-top:0.7rem; padding:0;">
  <div class="letter" style="border:none; border-radius:0;">
    <button class="copy-btn" data-copy-target="letter-text">Copy letter</button>
    <div id="letter-text">
      <!-- Header block, 3 paragraphs of narrative citing criteria, closing line -->
    </div>
  </div>
</div>
```

## Raw JSON shape

```json
{
  "verdict": "submit | needs_docs | do_not_submit",
  "reason": "one-sentence summary",
  "patient": { "name": "string", "dob": "YYYY-MM-DD" },
  "request": {
    "hcpcs": "string",
    "hcpcs_description": "string",
    "side": ["LT", "RT"],
    "icd10": ["string"],
    "prescriber": { "name": "string", "npi": "string" },
    "planned_dos": "YYYY-MM-DD"
  },
  "policy": {
    "id": "string",
    "title": "string",
    "in_scope_hcpcs_summary": "string",
    "requested_in_scope": true
  },
  "criteria_evaluated": true,
  "criteria_skipped_reason": null,
  "criteria_met": [
    { "section": "string", "finding": "string", "source_doc": "string", "source_date": "YYYY-MM-DD" }
  ],
  "missing_evidence": ["string"],
  "alternative_pathways": [
    { "path": "string", "policy": "string", "condition": "string", "action": "string" }
  ],
  "flags": ["string"],
  "draft_narrative": "string | null",
  "draft_word_count": 0,
  "confidence": 0.0
}
```

## Rules of thumb

- **Scope check before criteria check.** A code-out-of-scope verdict short-circuits the criteria evaluation; mark criteria as `N/E` (not evaluated) and put a note saying so. The user has saved hours of doomed drafting work the moment they see this verdict.
- **Narrative only appears when viable.** For DON'T SUBMIT, suppress the letter. Show "what to do instead" instead — numbered alternative pathways (different policy, ABN/self-pay, code rewrite).
- **Alternative pathways are specific.** "If diabetic → route to LCD L33369 (Therapeutic Shoes); if not → ABN + self-pay; if clinical intent is ankle stabilization → ask prescriber to rewrite as AFO (L1902/L1906)." Not generic.
- **Cite traceable sources.** Each criterion-met entry has a source doc + date. No invented LCD numbers, studies, or thresholds.
- **Copy-all on the Meta / Request card** with `.card-h-row` + hidden payload — multi-line block the appeals specialist can paste into their ticketing system.

## Reference example

`examples/sample-output.html` is the canonical example — Marie Curie's L3020 order reviewed against LCD L33686, verdict DON'T SUBMIT, with the three alternative pathways laid out (diabetic shoes / ABN self-pay / rewrite as AFO). Match that level of specificity in the verdict reasoning and the "what to do instead" actionability.

When the verdict turns out to be SUBMIT or NEEDS DOCS instead, also include the drafted narrative letter (header block + 3 paragraphs + closing) in a final `card.blue` below the criteria/evidence cards.
