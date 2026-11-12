# Open questions

Each entry says **what changes in the design depending on the answer**, so the list can be
prioritised rather than answered in order. Nothing below can be resolved from outside the
operation; none of it has been probed.

---

## A · The workflow itself

**A1. Does the eRx population still receive a faxed intake packet alongside the
e-prescription?**
The load-bearing one. If yes, the fax problem is an *intake-packet* problem and the model is
assembling a case file; if no, it is a *prescription* problem and the 30% is the whole
population of interest. Document taxonomy, schema, and whether cross-channel reconciliation
exists at all all follow from this.

**A2. What document types actually make up the fax population, and in what rough
proportions?**
Prescriptions, referral/enrollment forms, clinical notes, labs, insurance card images, PA
paperwork, and unrelated mail behave very differently. Pre-population screens and extraction
schemas are document-type specific, so this decides whether one pipeline or several.

**A3. What is the daily volume of true first-fill faxes — and of the whole fax queue?**
If first fills are a small slice of a large queue, triage (use case 1) is worth more than
extraction (use case 2), and the sequencing changes.

**A4. Where does a first fill actually begin today — an intake application, a case work
queue, or keyed straight into the pharmacy system?**
Decides what "pre-populate the intake screen" concretely means, and whether a human is even
looking at a screen that could be pre-filled.

**A5. What happens to a fax that cannot be acted on — is it parked, returned, or does
someone phone the prescriber's office?**
If the callback already exists as a workflow, the highest-value model output is a better
callback question, which is a much smaller build than integration.

---

## B · The system contract

**B1. Does the order API take a field-level payload, and is there a separate patient-create
or case-open endpoint?**
A field-level contract makes pre-population cheap and makes the human-commit step a review
screen. Screen-level automation makes it expensive and fragile, and changes the whole
cost model.

**B2. What does the first-fill rule API actually return, and is "no match" a first-class
outcome?**
If no-match is representable, the pipeline can abstain. If the API only returns a pharmacy
or errors, someone will write a default — and defaults are how first fills go wrong.

**B3. Can the patient/therapy book be queried by the identifiers that are actually
extractable — member ID, DOB, name plus address — and with what fuzzy tolerance?**
The first-fill-versus-refill classifier depends entirely on this lookup. No lookup means the
decision falls back to document appearance, which is the weakest possible signal.

**B4. Is an eRx feed available to the intake pipeline, and can a fax be joined to the same
patient or therapy?**
If yes, there is a genuine independent source and the source-error problem becomes
tractable. If no, every source error is a plausibility gate that can only flag, never
resolve.

**B5. Does the intake system already record a rejection or rework reason code?**
If it does, a defect taxonomy exists for free and the measurement plan gets much cheaper.

---

## C · Measurement

**C1. What is the current first-fill defect rate, and where do defects surface — pharmacist
verification, rejected claim, patient complaint, or audit clawback?**
This is the number the whole project is measured against. Without it, "streamlined" cannot
be shown to be an improvement.

**C2. What is the current time from fax receipt to first dispense, and what is the target?**
For oncology, MS, HIV and similar, a delayed first fill is a therapy gap, not an
inconvenience — so latency may be a better success metric than cost per fill.

**C3. Who reviews the packet — pharmacy technician, pharmacist, or prescriber-support rep —
and roughly how long does each take?**
Decides who the commit step belongs to, and whether pre-population is removing minutes or
seconds of work.

---

## D · Model and platform

**D1. Does the enterprise AI gateway expose per-field confidence or logprobs, or only
text?**
Without a usable score, the abstain threshold has to be built from cross-pass agreement
alone. Workable, but it changes the validator.

**D2. Can two different model vendors be reached through the gateway for the independent
second pass?**
Two passes from the same model share failure modes and buy much less than two vendors. This
is the difference between real corroboration and the appearance of it.

**D3. What are the retention and deletion rules for inbound fax images?**
Determines whether images can be kept for evaluation and audit, or must be discarded after
extraction — which in turn decides whether a labelled corpus can exist at all.

**D4. Is there any state or accreditation rule requiring a pharmacist to review the original
prescription image rather than a transcription?**
If yes, pre-population must remain advisory in those jurisdictions and cannot be the sole
basis of a dispense, regardless of measured accuracy.

---

## E · Risk ownership

**E1. Is ship-to-address change control reviewed for the first-fill path, and by whom?**
Shipment diversion is the highest-value attack against this workflow, and it is a control
question rather than a model question. If nobody owns it, the model is not the biggest risk
in the design.

**E2. What sign-off is required before a human may create an order from model pre-population
— and is it a different sign-off before anything is auto-submitted?**
The answer sets the realistic automation ceiling, and who has to agree to it.

**E3. Who owns the adversarial test corpus, and does it live long enough to be a regression
suite?**
A one-off red-team exercise decays. The control only holds if white-on-white text, margin
instructions, stamped-over fields and truncated packets are re-run on every model or prompt
change.