Re: Medical Necessity — HCPCS E0601 (CPAP) — DOS 2026-06-02
Patient: Jane R. Doe · DOB: 1971-04-22 · Member ID: 88291XJ4

This patient meets coverage criteria under **LCD L33718 §II.A.1.b**. Diagnostic polysomnography conducted on 2026-05-01 documented an Apnea-Hypopnea Index of **14.2 events/hour** with associated nocturnal hypoxemia (SpO2 nadir 84%) over a total sleep time of 387 minutes. This satisfies the threshold for obstructive sleep apnea with documented symptoms defined in §II.A.1.b (AHI ≥ 5 with documented symptomatic impact).

The treating physician's face-to-face evaluation, dated 2026-05-08, documents daytime hypersomnolence with an Epworth Sleepiness Scale of **15**, confirming the symptomatic impact required under §II.A.2. The evaluation occurred within 6 months of the sleep study and was performed by the treating provider, Dr. R. Patel, MD, satisfying §II.A.3. Comorbidities of note include hypertension and a BMI of 32. No prior trial of positive airway pressure therapy is documented.

CPAP therapy at the prescribed pressure of **9 cm H2O** with a nasal-pillow mask is reasonable and necessary for the management of this patient's moderate obstructive sleep apnea. Initiation is requested for date of service 2026-06-02.

— Draft 1 · 212 words · awaiting clinician sign-off

```json
{
  "code_requested": "E0601",
  "icd10": ["G47.33"],
  "payer": "BCBS MN",
  "policy_cited": "LCD L33718",
  "criteria_met": [
    {
      "section": "§II.A.1.b",
      "finding": "AHI 14.2 events/hour with SpO2 nadir 84%",
      "source_doc": "Polysomnography report",
      "source_date": "2026-05-01"
    },
    {
      "section": "§II.A.2",
      "finding": "Epworth Sleepiness Scale 15; daytime hypersomnolence",
      "source_doc": "Face-to-face evaluation",
      "source_date": "2026-05-08"
    },
    {
      "section": "§II.A.3",
      "finding": "F2F by treating provider within 6 months of sleep study",
      "source_doc": "Face-to-face evaluation",
      "source_date": "2026-05-08"
    }
  ],
  "missing_evidence": [],
  "physician_sign_off_required": true,
  "draft_word_count": 212,
  "confidence": 0.94
}
```
