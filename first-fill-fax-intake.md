# Faxed first-fill intake — use case and risk map

## What this is

Working notes on applying a vision-language model to faxed intake packets in specialty
pharmacy, written to settle the use case before any design commitments. It contains **no
architectural decisions** — deliberately, because the use case was misread once already and
the correction changed the shape of the problem.

Everything that describes a specific operation's internals is **reconstruction from
conversation, not observation**. It is marked where it matters.

## The correction that reframes the problem

An earlier read of "first fill pharmacy" treated it as a routing decision — which pharmacy
gets the first dispense. That is wrong in a specialty pharmacy that *is* the dispensing
pharmacy for the whole book: every first fill is the same pharmacy. "First fill" and
"refill" are two different **internal workflows**, and the boundary between them is
patient/therapy novelty, not how the document looks.

| | Refill | First fill |
|---|---|---|
| Patient record | exists | created |
| Prescription | on file, re-dispense | validated, first authorization |
| Benefits | adjudicated at least once | eligibility + coverage determination + copay assistance |
| Prior authorization | expiry check | full PA where required |
| Clinical | adherence check | pharmacist intake assessment, indication, labs |
| REMS | continue | certify pharmacy, prescriber, patient |
| Consents | on file | collected |
| Delivery | schedule | set up first shipment |

First fill is roughly an order of magnitude more work per fill than refill — an order of
magnitude, not a measurement. Most of that work is document- and data-bound, which is
exactly why it is the right place to automate, and why the fax matters more here than
anywhere else in the queue.

The consequence: the question is never *where does this go*. It is **is this a first fill,
and is the packet complete enough to start one.** Those are different questions with
different failure modes.

## Why a fifth to a third of fills never migrated to e-prescribing

A representative split in specialty pharmacy is roughly 70% of fills arriving by
e-prescription (NCPDP SCRIPT) and 30% still by fax, with higher support cost and higher
defect rates on the fax side. Hypotheses for why, **untested**:

- **SCRIPT carries the prescription message, not the intake packet.** Enrollment forms,
  clinical notes, labs, insurance card images, prior-authorisation documentation and
  prescriber attestations have no e-prescribing equivalent. If first fill needs the packet,
  eRx alone cannot retire the fax even where eRx already exists for the same patient.
- **Sender-side coverage is uneven** — small practices without an EHR, product categories
  their system cannot represent, REMS-only or compendium-specific products.
- **Contractual** — some programs still require a faxed record.

**The load-bearing question, before anything else:** does the 70% mean *fax-free*, or does
it mean *the prescription arrives by eRx and a faxed packet still arrives alongside it*? If
the second, the fax problem is an **intake-packet** problem rather than a prescription
problem, and the entire document taxonomy changes.

## What a vision-language model can do, ordered by value per unit of risk

1. **Triage.** Which queue the document belongs in — first fill, refill, PA-related, other.
   Highest volume, lowest stakes, reversible.
2. **Pre-populate the intake screen.** The model fills fields; a human confirms and submits.
   This is the sweet spot: most of the labour value, none of the autonomous-action risk, and
   every human edit becomes labelled ground truth for free.
3. **Source-error detection.** Flag internal inconsistencies and missing elements, and
   generate the precise callback question to the prescriber's office. Valuable at zero
   automation rate.
4. **Auto-submit to the order API.** Last, and only after (2) and (3) have produced a
   measured defect rate.

The point of that ordering: 1–3 need no autonomy and no trust in the model, and they still
remove most of the manual keying. Streamlining is reachable without ever letting a model
submit an order.

On (1): first-fill-versus-refill should ultimately be decided by a **lookup**, not by how the
document looks — does this patient/therapy already exist, and is this a transfer in. Document
appearance is a weak signal; a refill request can look identical to a new one.

## Three error sources that must never be conflated

| Source | Example | Fixed by better perception? |
|---|---|---|
| Model perception | reads `7` as `1`; drops a decimal point | Yes — second pass, pixel provenance |
| Adversarial | white-on-white text instructing a reroute | No — fixed by removing agency |
| **Source document** | office writes the NDC for 20 mg on an Rx for 10 mg | **No — it is perfectly legible** |

The third row is the trap. A wrong prescription transcribed with near-perfect accuracy is
still wrong, and the document is **self-consistent**: nothing inside the fax distinguishes it
from a correct one. A human reading the same page makes the same error, so a human review
step does not cover it either unless the human has a second source.

What catches source errors is reconciliation against something outside the fax:

- **Drug triangle** — name ↔ strength ↔ form ↔ NDC ↔ package must agree in a licensed drug
  file. Catches "right drug, NDC for the wrong strength", probably the most common fax defect.
- **Look-alike/sound-alike gate** — if the written name is near a known confusable pair and
  the strength matches the confusable partner, that is unresolved, not a guess.
- **Plan resolution** — BIN, PCN, group and member ID must all resolve to one plan. Also the
  gate that catches wrong-patient faxes.
- **Prescriber authority** — NPI ↔ DEA ↔ state licensure ↔ authority for that schedule or REMS
  product. Signature stamps and unauthorized delegates fail this deterministically.
- **Prescription validity** — signature present, date inside the state window, state-required
  elements present.
- **Sig arithmetic** — parse the directions, compute implied days supply, compare to written.
- **Duplicates and page completeness** — offices refax the same packet repeatedly; a 3-of-5
  packet must never advance.

Two small traps worth recording: at first fill **there is no Rx number yet** — that is
assigned by the pharmacy system, so any number on the fax is the sender's reference, not
identity. And the **fax CSID is not a sender identity**; anyone can set it, which is also how
a misdialled fax delivers one patient's PHI to the wrong pharmacy.

The discipline that matters most: **never auto-correct.** "The NDC is wrong but the name is
clear, so substitute it" is how something nobody authorized gets dispensed. Substitution must
be an explicit human correction, recorded, with the original preserved.

Operationally this means keeping **source-suspect** and **model-unsure** as distinct reason
codes on distinct queues. Merge them and both metrics become meaningless.

## Threat model

An inbound fax is unauthenticated, unattributable and attacker-controlled: anyone can send
one, and the header that claims to identify the sender is forgeable. With routing off the
table, what an attacker actually gains from a faxed document:

- **Shipment diversion** — altering the ship-to address on a first fill. The classic specialty
  diversion vector, and the highest-value one here.
- **Drug or strength substitution** on a created order.
- **A fabricated prescriber identity** that legitimizes an order.
- **PHI misdirection** — inducing an outbound document or callback to a wrong destination.
- **Queue poisoning** — burying a real first fill behind generated noise.

The control that carries the load does not depend on model quality: the model emits fields,
**code** calls the API, a human commits. Prompt hardening and injection detectors are
dampeners, not boundaries — worth deploying, never worth relying on. Putting the detector in
the same process and model as the transcriber also creates correlated failure, so it belongs
separately.

An enterprise AI gateway settles the data-flow and business-associate question. It does not
settle injection semantics, because injection is a correctness problem inside the model
boundary rather than a transport problem.

## Finding out where defects surface

A defect baseline is not needed to start. Deploying (2) above manufactures one: log every
field the human changes before submitting. That produces both a labelled corpus and an edit
rate per field, with nothing auto-submitted.

Worth instrumenting from day one: edit rate per field; callback volume to prescriber offices;
time from fax receipt to first dispense; rework reasons on rejected intake.

## Open questions

See [QUESTIONS.md](QUESTIONS.md) — each one is listed with what changes in the design
depending on the answer.

## Related

The logical architecture referenced above is [`architecture.html`](architecture.html).

## Evidence

Nothing in this note was probed from a live system. No workflow, queue, API or document was
inspected. Claims about a specific operation's first-fill workflow, the eRx/fax split and the
downstream API rest on conversation; the error taxonomy and the reconciliation controls are
general specialty-pharmacy reasoning and are marked as such. The 70/30 split is presented as
a typical figure, not a measured one.