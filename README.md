# First-fill fax intake

Design notes and a logical architecture for reading inbound faxes with a vision-language
model in a specialty-pharmacy **first fill** workflow.

First fill is the intake that establishes a patient on therapy: record creation, benefit
verification, prior authorisation, clinical assessment, REMS certification, consents, and the
first shipment. It is a different workflow from refill, and in a specialty pharmacy it is
where most of the manual document work sits — which is also why it is the interesting place
to point a model at.

## Contents

| File | What it is |
|---|---|
| [`architecture.html`](architecture.html) | Logical architecture — layers, responsibilities, workflow order. Single file, no dependencies; open in a browser. |
| [`first-fill-fax-intake.md`](first-fill-fax-intake.md) | The use case, why the fax still exists, the error taxonomy, and the threat model. |
| [`QUESTIONS.md`](QUESTIONS.md) | Open questions, each with what changes in the design depending on the answer. |

## The two ideas worth taking away

**The model transcribes; code decides; a human commits.** The vision model is the only
model in the pipeline and it has no tools, no network, no memory and no write access. It
emits a strict JSON object where every value carries a bounding box back to the source
pixels. Everything downstream — checksums, drug-file agreement, plan resolution, the
first-fill rule lookup, the order API call — is deterministic code. An instruction embedded
in a fax has nothing to actuate, because nothing in the model's reach can act.

**Three error sources, never conflated.** A misread digit (perception), text that tries to
instruct (adversarial), and a prescription that is perfectly legible and wrong (source
document) need three different controls and three different queues. The third is the one
people miss: the document is self-consistent, so no improvement in model accuracy touches
it, and a human reading the same page makes the same mistake. Only reconciliation against
something outside the fax catches it.

## Status

Draft, and honest about it. Nothing here was probed against a live system: no workflow, queue,
API or document was inspected. Claims about the specific operation come from conversation;
the domain reasoning is general. Every figure is labelled where it appears.

## Scope

Personal design notes. Not an official document of any employer or vendor, and not legal,
regulatory or clinical advice. No PHI, no patient data, no proprietary code.