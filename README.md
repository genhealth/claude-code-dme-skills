# Claude Code DME Skills

Three production-ready [Claude Code](https://claude.com/claude-code) skills for the back office of a DME or provider practice. Each one slots into the workflow your team already runs — fax intake, prior-auth review, outbound correspondence — and produces a polished, GenHealth-styled artifact (HTML dashboard or PDF letter) every time.

Open-sourced by **[GenHealth.ai](https://genhealth.ai)** — AI for the front office *and* the back office of healthcare delivery.

> **Important.** These skills produce drafts. A human (intake coordinator, biller, clinician, compliance officer) must review every output before any downstream action — claim submission, patient communication, or transmission to a referring office or payer. See `LICENSE` for the full disclaimer.

---

## The three skills

| Skill | What it does | Output |
|---|---|---|
| **`fax-intake-routing`** | Classifies + structures incoming faxes (PDFs, OCR text, packets) into routable work-queue tickets — patient, provider + NPI, HCPCS, ICD-10, attachments, urgency, target queue, confidence. | Single-screen HTML dashboard with copy buttons on every field. |
| **`prior-auth-drafting`** | *Reviews first, drafts second.* Checks the requested HCPCS against the payer policy scope and maps each clinical finding to each criterion. Verdict: **SUBMIT** / **NEEDS DOCS** / **DON'T SUBMIT.** Drafts the medical-necessity narrative only when the case is viable. | Single-screen HTML dashboard with verdict, criteria mapping, alternative pathways, and (conditional) the drafted letter. |
| **`draft-response`** | Drafts an outbound PDF letter to either the **referring physician** (pre-submission documentation request) or a **health plan** (PA cover, appeal, reconsideration, or coverage inquiry). User specifies recipient + purpose; skill picks the body shape. | Print-ready PDF with GenHealth attribution at top + bottom, supplier letterhead, and the appropriate body. |

Each skill's `SKILL.md` describes its workflow, required inputs, output schema, and tone rules in detail.

---

## Repo layout

```
claude-code-healthcare-skills/
├── README.md                           ← you are here
├── LICENSE                             ← MIT + healthcare-specific disclaimer
├── _shared/
│   ├── output-template.html            ← dashboard template (skills 1 & 2)
│   ├── letter-template.html            ← print/PDF letter template (skill 3)
│   └── genhealth-icon.svg              ← brand mark (used in templates + favicon)
├── fax-intake-routing/
│   ├── SKILL.md
│   └── examples/
│       ├── synthetic-fax.txt
│       └── sample-output.html       ← rendered dashboard for the synthetic fax
├── prior-auth-drafting/
│   ├── SKILL.md
│   └── examples/
│       ├── synthetic-input.json
│       └── sample-output.html       ← rendered DON'T SUBMIT verdict dashboard
└── draft-response/
    ├── SKILL.md
    └── examples/
        ├── referral-source-input.txt
        ├── payer-appeal-input.txt
        └── sample-output.html       ← rendered referral-source letter (Marie Curie case)
```

The `_shared/` folder holds two HTML templates that the skills read at runtime to produce their outputs. Don't move or rename it — the SKILL.md files reference paths inside `~/.claude/skills/_shared/`.

---

## Install

### Prerequisites

- macOS, Linux, or Windows (WSL)
- [Claude Code](https://claude.com/claude-code) installed and authenticated against an LLM endpoint with appropriate compliance posture (Amazon Bedrock + a signed BAA is the canonical path for PHI workloads — see the GenHealth webinar deck for the six-step setup)
- For `draft-response` only: a Chromium-based browser (Brave, Chrome, or Edge) installed at the standard macOS path so the skill can convert HTML → PDF via headless mode

### One-time setup

```bash
# 1) Clone somewhere stable
git clone https://github.com/genhealth/claude-code-healthcare-skills.git ~/code/claude-code-healthcare-skills

# 2) Symlink each skill folder + the shared assets into your global Claude Code skills directory
mkdir -p ~/.claude/skills
for d in fax-intake-routing prior-auth-drafting draft-response _shared; do
  ln -s ~/code/claude-code-healthcare-skills/$d ~/.claude/skills/$d
done

# 3) (Optional) verify
ls -la ~/.claude/skills/ | grep -E "fax-intake|prior-auth|draft-response|_shared"
```

Start a fresh Claude Code session and the skills will be available. You can invoke them directly (`/fax-intake-routing`, `/prior-auth-drafting`, `/draft-response`) or just describe what you want — the descriptions in each SKILL.md frontmatter route Claude to the right skill automatically.

### Per-project install (alternative)

If you'd rather scope the skills to a single project (and keep them out of your global Claude Code session), copy the folders into the project's `.claude/skills/` directory instead. Outputs will land in `./skill-output/` next to the project root.

---

## Try them

Each skill folder has an `examples/` directory with at least one synthetic input. The fastest way to confirm everything is wired up:

1. Open a Claude Code session in any project folder (so outputs have a place to land).
2. Paste the synthetic example and ask Claude to run the relevant skill — e.g.:
   - *"Route this fax: ..."* (paste `fax-intake-routing/examples/synthetic-fax.txt`)
   - *"Review this prior auth: ..."* (paste `prior-auth-drafting/examples/synthetic-input.json` along with the relevant LCD policy text)
   - *"Draft a documentation request to the referring office for this case: ..."* (paste `draft-response/examples/referral-source-input.txt`)
3. Watch the skill produce its output in `./skill-output/` and open it in your browser (or for `draft-response`, the PDF in your default PDF viewer).
4. Compare to the `sample-output.html` reference. (Open it in any browser — each sample is self-contained, no build step.)

All synthetic data is clearly marked. **Do not paste real PHI** into a Claude Code session that doesn't have a HIPAA-compliant backend (BAA + Amazon Bedrock or equivalent).

---

## Customize

The branding, attribution, and CTA lines live in two files — change them once and every output inherits the change:

- `_shared/output-template.html` — header chrome, copy-button JS, GenHealth CTA section, footer
- `_shared/letter-template.html` — print-ready letterhead, GenHealth attribution bars, footer disclaimer

Each `SKILL.md` describes the body structure the skill should produce — edit those for substantive changes (new fields, reordered cards, different copy patterns).

---

## Why GenHealth open-sourced this

These three skills are a glimpse of what AI can do at the front of a healthcare back-office workflow. **GenHealth's product does every one of these and more** — queue orchestration, EHR + clearinghouse integration, eligibility, billing, audit trails, and the compliance layer baked in from day one — built for DMEs and providers from day one, not a generic AI product retrofitted to healthcare.

If you want to see the full picture, [book a demo](https://genhealth.ai/demo) or [learn more at genhealth.ai](https://genhealth.ai).

---

## Contributing

Issues and PRs welcome. For substantive changes (new skills, new body structures, new modes for `draft-response`), open an issue first so we can talk through the shape.

If you ship one of these skills inside a real workflow and want to share what you learned — what worked, what broke, what edge cases tripped you up — we'd love to hear about it. Email [mike@genhealth.ai](mailto:mike@genhealth.ai).

---

## License

MIT — see `LICENSE`. The license includes a healthcare-specific disclaimer: this software does not constitute legal, medical, or compliance advice; outputs are drafts requiring human review; the authors do not guarantee HIPAA compliance or payer coverage outcomes.
