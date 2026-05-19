---
name: draft-response
description: Draft a polished, GenHealth-styled PDF response letter — to either the REFERRING PHYSICIAN (pre-submission documentation request) or a HEALTH PLAN / PAYER (prior-auth submission cover, appeal, reconsideration, or coverage inquiry). The user specifies recipient + purpose when invoking the skill; if it's unclear, the skill asks once. Outputs a print-ready PDF with GenHealth attribution at the top and bottom. Use when an intake review, prior-auth review, or denial requires an outbound letter. Triggers on phrases like "draft a response", "draft a letter", "draft an appeal", "draft a documentation request", "send the prescriber what's missing", "appeal this denial", "draft a payer letter", "respond to BCBS", "respond to referral source", "missing docs letter", "draft a PA cover", "draft a coverage inquiry".
---

# Draft Response

Generates a polished PDF letter to whoever needs to receive a response — the referring physician or the payer — using a single shared GenHealth letter template. Same letterhead, same look, same disclaimer; different body content per recipient + purpose.

## When to invoke

- Intake review found gaps → letter back to the referring office (`referral_source` · `documentation_request`)
- A prior-auth review came back DON'T SUBMIT due to a clinical-evidence gap → letter to the prescriber for the missing items (`referral_source` · `documentation_request`)
- A PA review came back SUBMIT or NEEDS DOCS → cover letter accompanying the submission to the payer (`payer` · `prior_auth_submission`)
- A claim was denied and you're filing an appeal (`payer` · `appeal`)
- You're asking the payer for coverage clarification (`payer` · `coverage_inquiry`)
- Any other case where a single outbound letter needs to be drafted from structured findings

## Recipient + purpose matrix

The user specifies one combination from this list when invoking. If they don't, ask once with a short menu.

| Recipient | Purpose | Body shape |
|---|---|---|
| `referral_source` | `documentation_request` | Intro · Coverage Outlook · Issues table · Numbered ask list · Next Steps |
| `payer` | `prior_auth_submission` | Intro · Patient + claim summary · Coverage criteria met (cited) · Enclosures list · Closing |
| `payer` | `appeal` | Intro · Original denial summary · Why the denial was inappropriate (policy + criteria + evidence) · Enclosures · Specific request · Deadline |
| `payer` | `reconsideration` | Reference to prior appeal · New / additional evidence · Specific request |
| `payer` | `coverage_inquiry` | Patient + plan context · Numbered questions · Request for written response by X date |

## HIPAA + testing notes

**Use synthetic data for testing.** Real PHI requires a signed BAA and HIPAA-compliant transport (fax to a secure number, payer portal, or prescriber portal).

The output is **always a draft.** An intake coordinator, biller, or supervisor signs before transmission.

In the chat reply, do NOT echo full letter content — report only: recipient, purpose, patient/case identifier, # of items addressed, PDF path.

## Required inputs

Always required:
- Sender (DME, medical group, or provider practice): name, address, intake/billing contact
- Patient identifiers: name, DOB, member ID (if known)
- Items ordered / service: HCPCS + description + side(s) + quantity
- Signer: title + contact

Additionally — for `referral_source`:
- Recipient prescriber(s): name + address + phone + fax
- Issues identified (list of `{ issue, impact }` pairs)
- Information requested (numbered list)
- Return-by deadline (default: 10 business days)

Additionally — for `payer`:
- Plan name + claims/appeals address (or payer portal)
- Claim ID (if appealing) + denial reason codes + denial date
- Policy reference (LCD or commercial policy section)
- Criteria met (list of `{ section, finding, source_doc, source_date }`)
- Specific request (e.g. "overturn the denial", "approve and pay claim DME-XXXX", "issue prior-auth approval for HCPCS L3020")

If anything is missing, ask once with a bulleted list and stop.

## Workflow

1. **Identify recipient + purpose.** If the user didn't say (or ambiguity exists), ask explicitly with a short numbered menu of the five combinations above.
2. **Verify inputs** for the chosen mode. Ask for missing items.
3. **Build the body HTML** matching the appropriate mode (see structures below).
4. **Read the template** at `~/.claude/skills/_shared/letter-template.html`.
5. **Substitute placeholders** with mode-appropriate values (see Placeholders).
6. **Ensure** `./skill-output/` exists.
7. **Write the HTML** to `./skill-output/<PatientLast>_<HCPCS>_<RecipientShort>_<Purpose>.html`. Examples:
   - `MarieCurie_L3020_PrescriberInfoRequest.html`
   - `JaneDoe_E0601_BCBS_Appeal.html`
   - `JohnSmith_K0823_Medicare_PASubmissionCover.html`
   - `LisaChen_L1902_Aetna_CoverageInquiry.html`
8. **Convert to PDF** via Brave headless:
   ```bash
   "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser" \
     --headless=new --disable-gpu --no-pdf-header-footer \
     --print-to-pdf="<absolute path>.pdf" "file://<absolute path>.html"
   ```
9. **Open** the PDF.
10. **In chat**, report (under 60 words): recipient + purpose, patient/case identifier, # items addressed, PDF path.

## Template placeholders

| Key | Meaning |
|---|---|
| `{{LETTER_TITLE}}` | Browser title — e.g. "Pre-Submission Documentation Request — Curie" or "Appeal of Claim DME-90412 — Doe" |
| `{{SENDER_NAME}}` | Sender name — DME, practice, or group (uppercased in letterhead) |
| `{{SENDER_ADDRESS_LINE}}` | Address • phone • fax — one line |
| `{{LETTER_DATE}}` | Long-form date |
| `{{RECIPIENT_NAMES}}` | "Dr. A / Dr. B" *or* "BCBS Minnesota — Appeals Dept" |
| `{{RECIPIENT_ADDRESS}}` | Street line |
| `{{RECIPIENT_CONTACT}}` | Phone/fax for prescriber; portal/fax for payer |
| `{{LETTER_RE}}` | RE: title — e.g. "Pre-Submission Documentation Request", "Appeal of Claim DME-90412", "Prior-Authorization Request — HCPCS L1902" |
| `{{RE_DETAILS}}` | Patient + DOB + DOS + items-ordered + claim ID (for appeals) — HTML allowed |
| `{{BODY}}` | Salutation + paragraphs + table/list (structure by mode below) |
| `{{SIGNER_LINE_1}}` | Signer title |
| `{{SIGNER_LINE_2}}` | Org line |
| `{{SIGNER_LINE_3}}` | Email + phone |
| `{{DISCLAIMER}}` | Footer disclaimer (mode-appropriate; defaults below) |

**Default disclaimers:**
- Referral source: *"This letter is a courtesy pre-submission notice and is not a payer denial. Coverage determinations are ultimately made by the insurer based on the submitted documentation and the patient's plan benefits. Contains PHI — handle in accordance with HIPAA."*
- Payer · appeal: *"This appeal is submitted in good faith based on documentation contained in the patient's record. Contains PHI — handle in accordance with HIPAA."*
- Payer · submission/inquiry: *"This correspondence accompanies the noted claim/authorization request. Contains PHI — handle in accordance with HIPAA."*

**Automatic in the template** (no skill action needed): the GenHealth attribution bar appears at both the top (above the sender letterhead) and the bottom (below the signature).

## Body structures by mode

### Mode 1 — `referral_source · documentation_request`

```html
<p>Dear Dr. &lt;Last&gt;:</p>
<p>Thank you for referring &lt;Patient&gt; for &lt;item&gt;. We have completed our intake review of the documentation you faxed on <strong>&lt;date&gt;</strong>. <strong>Before we deliver the device and submit a claim, we want to flag specific issues that, in our experience, will cause this order to be denied or returned by the patient's insurer.</strong></p>

<h2 class="section">Coverage Outlook</h2>
<p>... payer-specific policy notes ...</p>

<h2 class="section">Issues Identified in the Documentation Received</h2>
<table class="issues"><thead><tr><th>#</th><th>Issue</th><th>Impact</th></tr></thead><tbody>
  <!-- one row per issue -->
</tbody></table>

<h2 class="section">Additional Information Requested</h2>
<ol class="requests">
  <!-- one row per request -->
</ol>

<h2 class="section">Next Steps</h2>
<p>Return method + deadline + what happens next.</p>
```

### Mode 2 — `payer · prior_auth_submission`

```html
<p>To Whom It May Concern:</p>
<p>We are submitting prior authorization for <strong>&lt;Patient&gt;</strong> (Member ID &lt;ID&gt;) for <strong>HCPCS &lt;code&gt;</strong> (&lt;description&gt;), Date of Service <strong>&lt;DOS&gt;</strong>.</p>

<h2 class="section">Patient Summary</h2>
<p>... clinical context, diagnoses (ICD-10), prescribing physician + NPI ...</p>

<h2 class="section">Coverage Criteria Met</h2>
<ol class="requests">
  <li><strong>&lt;Policy §&gt;:</strong> &lt;criterion text&gt; — <em>&lt;clinical finding, source doc, date&gt;</em>.</li>
  <!-- one per criterion -->
</ol>

<h2 class="section">Documentation Enclosed</h2>
<ol class="requests">
  <li>Standard Written Order signed &lt;date&gt;</li>
  <li>Face-to-face evaluation &lt;date&gt;</li>
  <!-- etc -->
</ol>

<p>Please call our intake team with any questions or to expedite if appropriate.</p>
```

### Mode 3 — `payer · appeal`

```html
<p>To Whom It May Concern:</p>
<p>We are formally appealing the denial of claim <strong>&lt;claim_id&gt;</strong> for <strong>&lt;Patient&gt;</strong> (Member ID &lt;ID&gt;), Date of Service <strong>&lt;DOS&gt;</strong>, HCPCS <strong>&lt;code&gt;</strong>. The denial was issued on &lt;denial_date&gt; under reason code(s) <strong>&lt;codes&gt;</strong>.</p>

<h2 class="section">Why This Denial Should Be Overturned</h2>
<p>The denial cites &lt;reason&gt;. The patient's record demonstrates each criterion required under <strong>&lt;policy ref&gt;</strong>:</p>
<ol class="requests">
  <li><strong>&lt;§&gt;:</strong> &lt;criterion&gt; — &lt;matching clinical finding, source doc + date&gt;</li>
  <!-- one per met criterion -->
</ol>

<h2 class="section">Additional Documentation Enclosed</h2>
<ol class="requests">
  <!-- any extra evidence not in the original submission -->
</ol>

<h2 class="section">Specific Request</h2>
<p>We request that the denial of claim &lt;claim_id&gt; be <strong>overturned</strong> and the claim approved for payment in the amount of <strong>$&lt;amount&gt;</strong>. Please confirm in writing by &lt;deadline&gt;.</p>

<p>If additional information is required, please contact our appeals team at the number on the letterhead.</p>
```

### Mode 4 — `payer · reconsideration`

Similar to appeal, but explicitly reference the prior appeal: *"This is a second-level appeal following our initial appeal dated &lt;date&gt;, which was upheld on &lt;date&gt;. We are providing additional evidence..."* Then list the new evidence + restate the request.

### Mode 5 — `payer · coverage_inquiry`

```html
<p>To Whom It May Concern:</p>
<p>We are submitting a coverage inquiry for <strong>&lt;Patient&gt;</strong> (Member ID &lt;ID&gt;) regarding <strong>HCPCS &lt;code&gt;</strong>. We would like written confirmation on the following before scheduling delivery:</p>

<ol class="requests">
  <li>&lt;question 1&gt;</li>
  <li>&lt;question 2&gt;</li>
  <!-- numbered questions, specific -->
</ol>

<p>Please respond in writing within &lt;X&gt; business days.</p>
```

## Tone rules per mode

- **Referral source** — cooperative, "we want this to work." Plain English. Short paragraphs. No jargon.
- **Payer · submission / inquiry** — clinical, policy-citation-heavy. Quote section numbers. Use the exact words of the policy where possible.
- **Payer · appeal / reconsideration** — firm but professional. Evidence-driven. State a specific request and a deadline. No marketing language. No invented citations.

## Rules of thumb

- **No invented citations.** Only cite policies and study findings the user pasted. If they didn't paste the policy text, ask.
- **One letter, one purpose.** Don't combine documentation requests to a prescriber with appeals to a payer. Generate two separate PDFs if both are needed.
- **The signer signs.** Every PDF goes to a human before transmission.
- **Filename = recipient signal.** `_PrescriberInfoRequest.pdf` vs `_BCBS_Appeal.pdf` vs `_Medicare_PASubmissionCover.pdf` — make it obvious from the filename which kind of letter this is.

## Reference example

`examples/sample-output.html` is the canonical `referral_source · documentation_request` example — Boston Orthotics asking Drs. Raj/House for 10 specific docs before submitting Marie's L3020 order to BCBS. Match that level of specificity in the issues and asks. (The same content, converted to PDF by the skill, is what gets faxed.)

Other modes follow the same template (letterhead + RE block + body + signature + GenHealth bars top & bottom) with the appropriate body structure swapped in.
