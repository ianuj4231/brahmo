# BRAHMO Clinical AI â€” Final Architecture and Doctor-Readiness Plan

**Name:**  i.anuj

**Submission date:**  oct 21 

**Opening time:**  oct 21  . 2026...  12  pm 

**Submission time:**  oct 21 ... 2026...     2 pm  

## S1 Â· Final Architecture

### 1. Design goal

BRAHMO is a doctor-assistance system. It does not replace the doctor. It gathers evidence, identifies medicines, runs controlled safety checks and prepares a clear draft for the doctor to review.

The doctor remains responsible for the final clinical decision. The system never silently prescribes, diagnoses, calculates a patient-specific dose, or gives advice directly to a patient.

### 2. Main architecture

```text
Doctor enters a question or prescription
                  |
                  v
        Input and patient-context layer
                  |
                  v
       Structured clinical request created
                  |
        +---------+----------+
        |                    |
        v                    v
   Clinical question     Medicine input
        |                    |
        v                    v
   Evidence retrieval    Brand/alias resolution
        |                    |
        |              +-----+------+
        |              |            |
        |              v            v
        |        Clear identity   Unclear identity
        |              |            |
        |              v            v
        |       Safety rule engine  Ask doctor
        |              |            |
        +--------------+------------+
                       |
                       v
             Evidence-sufficiency gate
                       |
             +---------+----------+
             |                    |
             v                    v
       Enough evidence      Not enough evidence
             |                    |
             v                    v
      Grounded LLM draft     Honest abstention
             |                    |
             +---------+----------+
                       |
                       v
          Doctor reviews and decides
                       |
        +--------------+---------------+
        |              |               |
        v              v               v
      Approve        Edit            Reject/clarify
        |              |               |
        +--------------+---------------+
                       |
                       v
           Save/export result and audit
```

### 3. Step-by-step doctor journey

#### Step 1 â€” The doctor starts an interaction

The doctor asks a reference question or enters a prescription. Examples include:

- â€œWhat does the current national guidance say about this condition?â€
- â€œAre these two medicines using the same ingredient?â€
- â€œIs this combination product restricted?â€

The system records the consultation, doctor, organization and patient-context version. It does not treat the doctor's text as an instruction to change system rules.

#### Step 2 â€” The system gathers patient context

The patient-context service supplies a versioned snapshot, when available. The snapshot may contain age group, sex, pregnancy or lactation status, known allergies, current medicines and relevant conditions.

The system shows which fields are present, missing or stale. It does not invent missing patient information. If a required field is missing, the output says that the check is incomplete.

#### Step 3 â€” The system creates a structured request

The request is converted into structured fields:

```text
question text
requested answer type
medicines and strengths
formulation
patient-context reference
organization and language
consultation ID
```

The original text is kept for review. Structured extraction is an aid, not proof. If speech recognition, translation or extraction is uncertain, the uncertainty is shown.

#### Step 4 â€” The system identifies each medicine

For each medicine, the system follows this path:

```text
Entered brand or alias
        |
        v
Search exact names and known aliases
        |
        v
Check spelling, transliteration and formulation
        |
        v
Map to ingredient/salt, strength and form
        |
        v
Check whether the match is unique
        |
   +----+----+
   |         |
   v         v
Unique     Unclear
   |         |
   v         v
Continue   Ask doctor to select or clarify
```

Fuzzy matching may suggest possible matches. It cannot silently decide the final identity. A fixed-dose combination is split into its individual ingredients before safety checking.

#### Step 5 â€” The deterministic safety engine checks the prescription

The safety engine receives structured medicine identities and versioned data. It does not call the LLM.

```text
Resolved medicines
        |
        v
Check duplicate ingredients
        |
        v
Check combination ingredients
        |
        v
Check known interaction rules
        |
        v
Check allergy and population/form scope
        |
        v
Check regulatory events and restrictions
        |
        v
Check data freshness and service availability
        |
        v
Return a finding with evidence and status
```

Every check returns one explicit state:

```text
HIT                  = a rule found a relevant issue
CHECKED_NO_HIT       = checked coverage found no issue
PARTIAL_COVERAGE     = only part of the requested check was covered
UNVERIFIED_INPUT     = the input could not be trusted or identified
NOT_CHECKED          = this check was not applicable or not run
DATA_EXPIRED         = required data is too old
SOURCE_CONFLICT      = trusted sources disagree
SERVICE_UNAVAILABLE  = a required service was unavailable
```

No state other than `CHECKED_NO_HIT` means that the system has checked the relevant coverage and found no issue. Missing data must never appear as â€œsafe.â€

#### Step 6 â€” The evidence service retrieves sources

For a clinical reference question, the system searches the verified-live corpus:

```text
Doctor's question
        |
        v
Identify question type and scope
        |
        v
Apply filters: country, population, form, specialty, date
        |
        v
Run exact search for names, numbers and identifiers
        |
        v
Run semantic search for related wording
        |
        v
Rank and group evidence by source and version
        |
        v
Check applicability and sufficiency
```

Retrieval relevance is not enough. A source can look similar but apply to another population, formulation or country. The evidence gate checks whether the retrieved material actually supports the requested answer.

#### Step 7 â€” The system decides whether it may draft an answer

```text
Is the evidence sufficient and applicable?
        |
   +----+----+
   |         |
   v         v
   Yes       No
   |         |
   v         v
Create     Show what is missing,
grounded   conflicting or unverified
draft       |
   |         v
   |       Abstain or ask a clarification
   |
   v
Show claim-level citations
```

The LLM may interpret the doctor's wording and compose a concise answer from the approved retrieved evidence. It may not add unsupported clinical facts. Consequential claims must point to a source, version/date and source location.

#### Step 8 â€” The doctor reviews the result

The doctor sees:

- The answer in plain language
- The sources supporting each important claim
- The source date and scope
- Medication findings and their status
- Missing or conflicting information
- Any clarification request
- A visible label when information is international, local or background-only

The doctor may approve, edit, reject or request clarification. Nothing consequential is finalized automatically.

#### Step 9 â€” The system saves the result

After the doctor acts, the system stores the approved output or rejection and the audit information:

```text
request and patient-context version
medicine-resolution results
retrieved source IDs and versions
drug-data and rule versions
model and prompt versions
system status and warnings
doctor action and final text
time and organization
```

The saved record can later be replayed to understand what the system knew and what it showed at that time.

### 4. Reference-question path

```text
Doctor asks: â€œWhat does the selected guideline say?â€
        |
        v
Classify as reference question
        |
        v
Find applicable verified sources
        |
        v
Check population, specialty, country and date
        |
        v
Compose short answer from retrieved evidence
        |
        v
Attach claim-level citations
        |
        v
Doctor reviews and approves or edits
        |
        v
Save answer and audit trace
```

If the source cannot be found or its scope is unclear, the system says: â€œI cannot confirm this from the currently verified sources,â€ and identifies the missing evidence.

### 5. Medication-review path

```text
Doctor enters medicine names
        |
        v
Resolve each brand to ingredient, strength and form
        |
        v
Ask for clarification if any identity is uncertain
        |
        v
Decompose combination products
        |
        v
Run deterministic checks
        |
        v
Show finding, evidence, scope and status
        |
        v
Doctor decides what to do
        |
        v
Save the decision and audit trace
```

Example doctor-facing output:

```text
Medicine identity: Confirmed from the approved drug master.
Ingredients: [ingredient records shown here].
Duplicate-ingredient check: CHECKED_NO_HIT.
Restriction check: PARTIAL_COVERAGE.
Reason: the required population-specific source is not available.
Action: do not display this as â€œsafeâ€; ask the doctor to review the cited source or clarify the patient context.
```

### 6. Main components and boundaries

| Component | Responsibility | Must not do |
|---|---|---|
| Doctor-facing application | Collect input, show evidence and statuses, capture doctor action | Make an invisible clinical decision |
| Scribe adapter | Accept structured entities and transcript references | Treat transcript text as trusted instructions |
| Patient-context adapter | Supply a versioned context snapshot | Invent missing patient facts |
| Drug master | Resolve brands, aliases, ingredients, strengths and forms | Use stock presence as proof of legal status |
| Regulatory event store | Store effective-dated status events and scope | Reduce status to one permanent banned flag |
| Safety rule engine | Run deterministic medication checks | Call an LLM or guess missing data |
| Evidence pipeline | Acquire, verify, version and publish sources | Publish unreviewed extraction |
| Retrieval service | Find relevant, applicable evidence | Treat similarity as proof |
| Orchestrator | Route requests and combine structured results | Override safety findings |
| LLM service | Interpret questions and compose grounded drafts | Decide safety, identity or regulatory truth |
| Audit service | Record versions, actions and replay information | Allow unauthorized access |

### 7. Trust boundaries and important interfaces

Untrusted inputs include doctor text, transcripts, uploaded documents, imported stock data and organization documents. They are treated as data, not as system instructions.

The most important interfaces are:

```text
Scribe â†’ structured clinical entities + transcript references
Patient-context service â†’ versioned patient snapshot
Drug master â†’ resolved ingredient/form/scope record
Safety engine â†’ finding + required status + evidence references
Retrieval service â†’ ranked evidence + applicability metadata
LLM service â†’ draft text constrained to supplied evidence
Doctor interface â†’ approve/edit/reject/clarify action
Audit service â†’ immutable versioned event trail
```

Each interface must validate its schema, version and authorization. A malformed or incomplete response becomes a visible degraded state, not a silent fallback.

### 8. Orchestration state machine and tool restrictions

The orchestration layer is a controlled state machine. A framework such as LangGraph could implement this design, but the safety behavior does not depend on that particular framework.

```text
RECEIVED
   |
   v
CLASSIFIED
   |
   +--> CLARIFICATION_REQUIRED
   |
   v
CONTEXT_READY
   |
   +--> CONTEXT_UNAVAILABLE
   |
   v
IDENTITY_RESOLVED
   |
   +--> IDENTITY_AMBIGUOUS
   |
   v
CHECKS_COMPLETE
   |
   +--> CHECKS_DEGRADED
   |
   v
EVIDENCE_READY
   |
   +--> EVIDENCE_INSUFFICIENT
   |
   v
DRAFT_READY
   |
   v
DOCTOR_REVIEW
   |
   +--> EDITED / REJECTED / CLARIFICATION_REQUIRED
   |
   v
APPROVED_AND_AUDITED
```

Each transition has a condition and an allowed next state. The model cannot skip from `RECEIVED` to `APPROVED_AND_AUDITED`, and it cannot move to `DRAFT_READY` when identity, evidence or required checks are unresolved.

Tool access is allowlisted by state and role:

| State | Allowed tools | Prohibited tools/actions |
|---|---|---|
| `RECEIVED` / `CLASSIFIED` | Input parser and question classifier | Patient-record write, safety verdict, regulatory update |
| `CONTEXT_READY` | Authorized patient-context read | Cross-tenant read, patient-context write without explicit workflow |
| `IDENTITY_RESOLVED` | Drug-master read and ambiguity queue | LLM-only identity approval, direct drug-master mutation |
| `CHECKS_COMPLETE` | Deterministic rule engine and versioned data reads | LLM safety decision, prescription finalization |
| `EVIDENCE_READY` | Scoped retrieval and citation resolver | Unapproved source retrieval, source publication |
| `DRAFT_READY` | Grounded composition model | Sending patient advice, changing rules or permissions |
| `DOCTOR_REVIEW` | Display, edit, approve, reject and correction capture | Automatic clinical approval |
| `APPROVED_AND_AUDITED` | Save/export and audit append | Silent overwrite or deletion of the audit trail |

The LLM receives only the tools and evidence allowed for the current state. Tool calls are validated by the orchestrator, not accepted merely because the model requested them. Documents and transcripts are data inside a state; instructions embedded in them cannot change the state machine, tool allowlist, system policy or tenant authorization.

Every transition records its pre-state, post-state, condition result, actor, tool call, versions and failure reason. Invalid transitions are denied and logged. This makes the golden-thread test, prompt-injection test and audit replay testable.

## S2 Â· Decision Register

The proposed architecture keeps the doctor as the final decision-maker. The LLM assists with grounded retrieval and drafting, while identity resolution, regulatory status and medication-safety checks use versioned deterministic data and rules.

### D1. Medication-safety checks must remain outside the LLM

**Current position or omission:**  
The supplied architecture requires deterministic medication-safety checks, but the exact boundary between the language model and the safety system needs to be made explicit.

**My choice:**  
All medication-safety checks will run outside the LLM using versioned drug data and deterministic rules. The LLM may explain a returned finding only when the finding includes evidence and provenance.

**Why:**  
Medication checks must be repeatable, testable and replayable. An LLM can hallucinate, omit an ingredient or interpret the same prescription inconsistently. A missing or unsupported input must produce an explicit unknown state, never a safe result.

**Strongest rejected alternative:**  
Allow the LLM to interpret the prescription and directly decide whether it is safe.

**Accepted trade-off:**  
The system will have narrower initial coverage and will require curated drug data, clinical review and ongoing rule maintenance. This is preferable to presenting unreliable safety conclusions.

**How the choice will be validated:**  
Use held-out cases covering duplicate ingredients, restricted or banned combinations, interactions, missing information, stale data, ambiguous brands and unavailable services. Confirm that every result uses the required eight-state status contract and that no incomplete check renders as safe.

### D2. Brand names must resolve to ingredients before checking

**Current position or omission:**  
The supplied design identifies an alias-to-brand-to-salt graph and FDC decomposition, but the clinical importance of making ingredient-level identity the safety boundary must be explicit.

**My choice:**  
Every medicine is resolved through a versioned graph to ingredient or salt, strength, dose form and relevant population/formulation scope before any medication check. Fixed-dose combinations are decomposed before checking. Ambiguous resolutions go to a clarification queue and are never silently auto-resolved.

**Why:**  
Indian prescribing and stock records may contain brand variants, spelling errors, transliteration and combination products. A brand string alone is not a reliable safety identity, and two products with similar names may contain different ingredients.

**Strongest rejected alternative:**  
Use fuzzy matching to select the closest brand and proceed automatically.

**Accepted trade-off:**  
Some consultations will require a doctor clarification, creating friction and reducing apparent automation. That cost is acceptable because a wrong ingredient mapping can invalidate every downstream safety result.

**How the choice will be validated:**  
Maintain an independently held-out mapping set containing spelling variations, regional aliases, transliterations, look-alike names, different strengths, formulations and FDCs. Release requires zero silent wrong mappings; uncertain cases must be classified with a reason code.

### D3. Consequential answers require claim-level evidence

**Current position or omission:**  
The supplied plan requires citations, but the answer must also show which source supports each consequential claim rather than displaying a general list of references.

**My choice:**  
Every consequential claimâ€”such as a dose reference, numeric threshold, safety statement or regulatory statusâ€”must link to a versioned source, date and location. If suitable evidence is not retrieved or applicability is uncertain, the system states that it cannot confirm the claim.

**Why:**  
Doctors need to distinguish supported information from generated wording. Claim-level provenance also makes review, correction and audit possible.

**Strongest rejected alternative:**  
Display citations only at the end of the answer or rely on the model's general medical knowledge.

**Accepted trade-off:**  
Answers may be shorter, slower or occasionally abstain even when the model could produce a plausible response. This protects against unsupported clinical confidence.

**How the choice will be validated:**  
Use an evaluation set with independently verified claim-to-source links. Test uncited consequential claims, incorrect source locations, superseded sources, conflicting sources and evidence that is relevant but outside the requested population or formulation.

### D4. Retrieval should be hybrid and scope-aware

**Current position or omission:**  
The supplied design proposes hybrid retrieval and decision-unit chunks, but retrieval must preserve clinical scope and must not treat semantic similarity as proof of applicability.

**My choice:**  
Use a hybrid search combining exact terms, metadata filters and semantic retrieval. Index structured decision units with source title, section, population, formulation, specialty, jurisdiction, effective date and status. A deterministic evidence-sufficiency gate must run before generation.

**Why:**  
Exact matching helps with medicine names, numbers and regulatory identifiers. Semantic retrieval helps with varied wording. Metadata prevents a visually similar passage for the wrong population or formulation from being treated as applicable.

**Strongest rejected alternative:**  
Use only vector similarity over large text chunks.

**Accepted trade-off:**  
The ingestion and indexing pipeline becomes more complex, and some questions will abstain when scope cannot be established. The benefit is safer and more inspectable retrieval.

**How the choice will be validated:**  
Measure retrieval recall separately from applicability and citation correctness. Test paraphrases, multilingual or transliterated terms, table content, units, exceptions, footnotes, cross-page references, superseded editions and concurrent source conflicts.

### D5. Drug data should use a canonical master with separate overlays

**Current position or omission:**  
The supplied plan proposes a drug-master spine and pharmacy overlays, but the authority boundary between canonical drug truth and local stock information needs to be explicit.

**My choice:**  
Maintain a versioned canonical drug master for ingredients, strengths, forms, FDC composition and regulatory events. Store pharmacy, hospital and clinic data as separate overlays that may add local aliases, availability and price information but may not redefine canonical identity or regulatory truth.

**Why:**  
A medicine can be available in stock while restricted, expired, incorrectly labelled or unsuitable for a particular scope. Separating the data prevents local inventory from becoming an accidental authority.

**Strongest rejected alternative:**  
Treat the clinic or pharmacy inventory as the main source of medicine identity and availability truth.

**Accepted trade-off:**  
The system must reconcile multiple datasets and may show a visible disagreement instead of one convenient answer. Local data will also require its own freshness and permission controls.

**How the choice will be validated:**  
Test products that are in stock but restricted, products with different formulations under similar names, changed regulatory events, expired local records and conflicting canonical and overlay data. Confirm that the output labels the source and does not merge conflicts silently.

### D6. Use a managed general-purpose model only as a constrained composition layer

**Current position or omission:**  
The current approach uses a frontier LLM, but the system must define what the model is allowed to do and how provider changes are controlled.

**My choice:**  
Use a supported model for question interpretation, summarisation and grounded response composition. The model must not be the authority for drug identity, medication-safety verdicts or regulatory status. Prompts, model versions, tools and output schemas will be pinned and included in the release manifest.

**Why:**  
This gives useful language capability without allowing generated text to become the source of clinical truth. It also makes model changes observable and reversible.

**Strongest rejected alternative:**  
Train or fine-tune a custom medical model and allow it to answer directly from its parameters.

**Accepted trade-off:**  
The product depends on a provider and may have latency, cost and availability constraints. The system also needs stronger grounding and regression testing than a general chatbot.

**How the choice will be validated:**  
Run held-out tests after every model, prompt or tool change. Check citation faithfulness, unsupported claims, refusal behavior, prompt injection, cross-tenant leakage and consistency of structured outputs. A provider outage must produce a visible degraded state.

### D7. Patient data must be tenant-isolated and processed for a defined purpose

**Current position or omission:**  
The supplied plan mentions tenants and PHI-in-logs scanning, but it needs a concrete privacy and isolation decision.

**My choice:**  
Patient context, transcripts, corrections and audit records will be scoped by organization, user role, patient and consultation. Access will use least privilege, tenant-aware authorization and encryption in transit and at rest. Production patient data will not be used to train a provider model unless a separately approved lawful basis and contract permit it.

**Why:**  
Shared devices and multiple clinics create a real risk of exposing one patient or organization's information to another. Purpose limitation also reduces accidental reuse of clinical data.

**Strongest rejected alternative:**  
Use one shared searchable patient index with application-level filtering added later.

**Accepted trade-off:**  
Tenant-aware indexing, authorization checks and testing increase implementation complexity and may limit some global analytics. That cost is necessary for confidentiality.

**How the choice will be validated:**  
Use negative authorization tests for every tenant and role, shared-device session tests, export tests, deletion and retention tests, log-scanning tests and adversarial attempts to retrieve another patient's context through prompts or search terms.

### D8. The audit record must support exact replay

**Current position or omission:**  
The supplied plan requires replayability, but the minimum replay record must be defined.

**My choice:**  
For every consequential output, record the request and patient-context version, retrieved evidence identifiers and versions, drug-data and rule versions, model and prompt versions, tool results, input ambiguity, output, doctor edits or approval, timestamps and service status. Sensitive values should be minimized or protected according to the approved retention policy.

**Why:**  
A later reviewer must be able to understand what the system knew, what it did, what it displayed and what the doctor accepted. This is essential for incidents, corrections and release comparison.

**Strongest rejected alternative:**  
Keep only the final answer and a general application log.

**Accepted trade-off:**  
Audit storage, privacy controls and replay tooling add operational cost and require careful handling of sensitive records. The audit must not become an uncontrolled duplicate patient database.

**How the choice will be validated:**  
Reconstruct golden-thread consultations byte-for-byte where applicable and semantically where provider nondeterminism is unavoidable. Test missing events, tampered events, version mismatches, retention expiry and unauthorized audit access.

### D9. Degraded operation must be visible and capability-specific

**Current position or omission:**  
The product context includes unreliable connectivity, but offline behavior must not make unavailable evidence appear current or complete.

**My choice:**  
When retrieval, drug data, patient context or audit services are unavailable, the interface shows the affected capability and its exact status. Cached content is read-only, labelled with its version and freshness, and cannot silently substitute for current regulatory or safety data. Consequential checks return `SERVICE_UNAVAILABLE`, `DATA_EXPIRED` or another applicable non-safe state.

**Why:**  
Doctors may work on shared machines or unreliable networks. A visible limitation is safer than a normal-looking answer based on stale or incomplete information.

**Strongest rejected alternative:**  
Use the last successful answer or an unrestricted local model as a quiet fallback.

**Accepted trade-off:**  
Some functionality will stop or become reference-only during outages. This can interrupt workflow, but it avoids false confidence.

**How the choice will be validated:**  
Inject failures for network, retrieval, drug master, regulatory data, patient context and audit services. Confirm correct banners, non-safe statuses, no silent fallback, correct recovery behavior and complete recording of the degraded event.

### D10. Organization protocols are scoped overlays, not universal truth

**Current position or omission:**  
The supplied plan allows tenant protocols to outrank national defaults for that tenant, but it needs clear limits and conflict behavior.

**My choice:**  
An organization protocol may be used only for that organization's users and patients, with an explicit tenant label, owner, approval status, scope and effective date. It cannot override a global safety or regulatory prohibition. Genuine conflicts with a national or higher-authority source are shown and queued for review rather than silently merged.

**Why:**  
Hospitals may have valid local formularies or workflows, but a local document should not become a national clinical or legal fact. Clear labels preserve doctor understanding.

**Strongest rejected alternative:**  
Let each organization replace the default corpus without visible precedence or conflict handling.

**Accepted trade-off:**  
Doctors may see more than one applicable source and may need to resolve a disagreement. Governance and review queues become more complex, but the system remains honest about scope.

**How the choice will be validated:**  
Test tenant isolation, overlay precedence, expired approvals, conflicting protocol versions, attempts to override safety blocks and answers generated for users who do not belong to the organization.

### D11. LLM-assisted extraction is allowed only before verification

**Current position or omission:**  
The supplied plan describes raw, structured, verified and live knowledge tiers, but it does not clearly define whether an LLM may be used to extract information from raw documents.

**My choice:**  
Use deterministic parsers, OCR and table extraction first. An LLM may assist with difficult extraction and normalization as an offline drafting tool, producing a structured candidate record with source location and extraction provenance. The candidate must pass schema and consistency checks and qualified human review before entering the verified-live corpus.

**Why:**  
Clinical documents contain headings, tables, units, exceptions, footnotes and cross-page relationships that simple text splitting may damage. LLM assistance can improve extraction of difficult structure, but it must not be allowed to turn an unverified guess into clinical truth.

**Strongest rejected alternative:**  
Either use the LLM as the final authority during ingestion or prohibit all model use and rely only on basic text extraction. The first is unsafe; the second may lose important document structure and increase manual effort unnecessarily.

**Accepted trade-off:**  
The pipeline becomes more complex because every extracted record needs provenance, automated checks and human review. Extraction coverage may also be slower when review queues are full. Unreviewed content will remain unavailable for consequential answers.

**How the choice will be validated:**  
Compare extracted records against clinician-reviewed reference records for ingredients, strengths, units, populations, formulations, exceptions, tables and citations. Seed errors such as dropped decimal points, wrong units, merged rows and lost footnotes. Confirm that no unverified record can be retrieved by the live clinical answer service.

**Boundary:**  
This LLM use is limited to the offline knowledge-ingestion lane. No LLM call is permitted inside the deterministic medication-safety check path.

### D12. Use an explicit state machine and allowlisted tool routing

**Current position or omission:**  
The supplied architecture names orchestration and structured-first routing, but it does not fully specify how the system prevents an LLM or untrusted document from skipping a safety step or calling an inappropriate tool.

**My choice:**  
Implement orchestration as an explicit state machine with conditional branches. Each state has an allowlist of tools and permitted next states. LangGraph or a similar workflow framework may implement it, but the state transitions, authorization and safety rules remain application-owned.

**Why:**  
Explicit states make it possible to prove that ambiguous identity, incomplete checks, insufficient evidence and doctor approval cannot be silently bypassed. Tool restrictions also reduce prompt-injection impact.

**Strongest rejected alternative:**  
Let the LLM choose tools and sequence the workflow dynamically from a general system prompt.

**Accepted trade-off:**  
The workflow is less flexible and requires more state definitions, transition tests and maintenance. This is acceptable for a consequential clinical workflow.

**How the choice will be validated:**  
Test every valid and invalid transition, tool call outside its allowlist, prompt injection in documents, cross-tenant tool access, service failures and attempts to reach approval without doctor review. Replay must show the complete state and tool history.

### D13. Use section-aware decision-unit chunks with linked context

**Current position or omission:**  
The supplied plan rejects bulk token chunking, but it does not define how to preserve meaning across headings, tables, exceptions and pages.

**My choice:**  
Create chunks at natural clinical boundaries: one recommendation, rule, definition, table row group or exception with its heading, scope, units and citation. Store a smaller searchable child chunk linked to a larger parent section. Retrieve the child for ranking, then attach the required parent context before generation. Use token limits only as a final safety limit.

**Why:**  
A small chunk improves search precision, while parent context prevents the LLM from seeing an isolated row without its table heading, population or footnote.

**Strongest rejected alternative:**  
Split every document into equal token windows with overlap.

**Accepted trade-off:**  
The ingestion pipeline needs more document-aware parsing, chunk links and review. Retrieved context may be slightly larger because required parent context is included.

**How the choice will be validated:**  
Test whether each chunk preserves the recommendation, heading, table columns, units, exceptions, footnotes and cross-page links. Compare child-only retrieval with child-plus-parent retrieval for recall, applicability, citation support and confidently wrong answers. Tune chunk size and parent context on development data and confirm on held-out cases.

## S3 Â· Grounding & Sources Plan

### 1. Source strategy

BRAHMO should not treat one source as correct for every question. Each source has a purpose, scope and authority. The system stores source metadata and shows it to the doctor.

| Source family | Recommended use | Authority and scope | Version/date | Rights and limitation |
|---|---|---|---|---|
| [ICMR Standard Treatment Workflows](https://www.icmr.gov.in/standard-treatment-workflows-stws) | Indian clinical reference answers and specialty workflows | Government clinical guidance organised by specialty and condition | Use the edition/date printed on each workflow; the catalogue currently lists volumes and specialty documents | Access is public, but storage, extraction, indexing and displayed excerpts must be checked against the document's copyright and reuse terms. It is not universal coverage of every condition. |
| [ICMR guidelines](https://www.icmr.gov.in/guidelines) | Topic-specific Indian guidance, such as antimicrobial-use guidance | ICMR guidance for the stated topic | Store publication and revision date | Public access does not automatically prove permission for unrestricted reproduction or training. Record permission status per document. |
| [Central TB Division / NTEP guidelines](https://tbcindia.mohfw.gov.in/guidelines/) | National TB programme questions and TB-specific workflows | Programme authority for its stated TB population, setting and regimen | Use the exact document title, issue date and supersession status; do not use an old guideline when a newer one applies | Public government access; verify reuse and excerpt-display rights. Scope is TB-specific and must not be generalised to unrelated conditions. |
| [MoHFW National Immunization Schedule](https://www.mohfw.gov.in/sites/default/files/Unit2NationalImmunizationSchedule.pdf) | Reference information about the national immunization schedule | MoHFW programme material for the stated population and schedule | Store the PDF date and review the current MoHFW version before serving | Use as a reference source, not as permission to add patient-specific dose calculation or a patient-facing advice surface. |
| [CDSCO Essential Medicines](https://cdsco.gov.in/opencms/opencms/en/consumer/Essential-Medicines/) and [NLEM 2022](https://cdsco.gov.in/opencms/opencms/en/consumer/Essential-Medicines/) | Essential-medicine classification and reference | Government drug-information source for the published list | NLEM 2022 is a dated edition; newer editions must supersede it when officially published | It does not by itself establish that a particular brand is legal, available or suitable for a patient. Verify rights and intended use. |
| [CDSCO notifications](https://cdsco.gov.in/opencms/opencms/en/Notifications/Gazette-Notifications/) | Regulatory events, notifications and effective dates | Primary regulatory material for the event's scope | Store notification ID, publication date, effective date, supersession and affected products/substances | A public notification may be viewed, but storage, extraction and displayed excerpts need a rights decision. Regulatory interpretation requires qualified review. |
| Approved licensed drug data | Ingredients, strengths, forms, interaction and pregnancy/lactation rule inputs | Licensed or contractually curated source, reviewed by a qualified clinical pharmacologist | Version and effective date supplied by the licensor/curator | A source must explicitly permit storage, extraction, indexing, display and any permitted derivative data. Access is not the same as permission. |
| Organization protocols | Local formulary, workflow or approved protocol for that organization | Tenant-scoped authority only | Owner, approval date, effective date and expiry required | May not override a global safety or regulatory prohibition. Rights and tenant membership must be recorded. |

No source is described as â€œcompleteâ€ without an evidence and rights review. International sources may provide background context, but they are labelled as non-India sources and do not silently outrank an applicable Indian source.

### 2. Source precedence

Precedence is determined by applicability, authority, effective date and scopeâ€”not by a permanent source ranking.

```text
Find candidate sources
        |
        v
Filter by country, population, specialty and formulation
        |
        v
Remove withdrawn or superseded material
        |
        v
Prefer the applicable Indian programme/regulatory source
when it directly covers the question
        |
        v
Use an approved organization protocol only for that tenant
        |
        v
Use international guidance for background or where Indian
coverage is absent, with an explicit label
        |
        v
If applicable sources still conflict, show the conflict and abstain
from silently choosing one
```

â€œSupersededâ€ means an older edition has been replaced. â€œConflictâ€ means two applicable sources are concurrently inconsistent. These states must not be merged.

### 3. Knowledge lifecycle

```text
Acquire permitted source
        |
        v
Record source identity, owner, rights, version and date
        |
        v
Extract text, tables and document structure
        |
        v
Create candidate structured records with page/section locations
        |
        v
Run schema, unit, reference and cross-record checks
        |
        v
Qualified reviewer checks clinical meaning and applicability
        |
        v
Chunk into decision units and create search indexes
        |
        v
Publish only approved records as verified-live
        |
        v
Retrieve only eligible, current and in-scope records
        |
        v
Monitor updates, conflicts, corrections and rights expiry
        |
        v
Withdraw, supersede or republish with a new version
```

An LLM may assist extraction in the candidate stage, as described in S2 D11. It cannot approve content or publish directly to verified-live.

### 4. Chunking design

The system uses clinical decision-unit chunks rather than arbitrary token windows. A chunk should normally contain one complete recommendation, definition, table row group or safety rule together with the context needed to understand it.

The preferred structure is **child chunk + parent context**. The child is small enough for accurate retrieval. The parent supplies the section heading, table title, population, formulation, exceptions and footnotes needed for safe interpretation. If the parent context cannot be attached, the result is not eligible for a consequential answer.

Each child chunk stores a link to its parent:

```json
{
  "chunk_id": "drug-guide-001-child-04",
  "parent_id": "drug-guide-001-section-07",
  "document_id": "drug-guide-001",
  "document_version": "2026-01",
  "chunk_type": "table-row-group",
  "section": "Adult formulation guidance",
  "page": 12,
  "population": "adult",
  "formulation": "tablet"
}
```

The embedding index searches the child chunk. The retrieval service uses `parent_id` to fetch the parent heading, table context, exceptions and footnotes before the evidence-sufficiency check. The parent and child versions must match.

There are two separate moments:

**A. Offline source preparation — before the doctor asks anything**

```text
Clinical guideline or document arrives
        |
        v
Extract text, tables, headings and footnotes
        |
        v
Create reviewed chunks and source metadata
        |
        v
Create exact, metadata and embedding indexes
        |
        v
Publish approved chunks to the verified-live corpus
```

This happens when a source is first added or updated. The system does not re-read and re-extract the whole document for every doctor question.

**B. Live doctor question — during consultation**

```text
Doctor asks a question
        |
        v
Search the prepared indexes
        |
        v
Retrieve relevant chunks
        |
        v
Check scope, applicability and evidence sufficiency
        |
        v
Send only the validated chunks to the LLM
        |
        v
Show the doctor a cited draft
```

The first flow prepares the knowledge. The second flow uses that prepared knowledge. A new or changed source repeats the first flow before it can be used in the second flow.

### 4a. Who does each preparation step?

```text
Source owner or legal reviewer
        |
        v
Confirms source identity, permission and intended use
        |
        v
Data-ingestion engineer/service
        |
        v
Downloads source and extracts text, tables and metadata
        |
        v
Extraction pipeline + optional LLM assistant
        |
        v
Creates candidate chunks; does not approve them
        |
        v
Data QA reviewer/service
        |
        v
Checks schema, units, missing rows, page links and duplicates
        |
        v
Qualified clinical reviewer
        |
        v
Checks clinical meaning, scope, exceptions and applicability
        |
        v
Regulatory/legal reviewer, when relevant
        |
        v
Checks regulatory interpretation and source-rights restrictions
        |
        v
Knowledge owner
        |
        v
Publishes the approved version to verified-live
        |
        v
Platform/retrieval engineer
        |
        v
Builds the exact, metadata and embedding indexes
```

Responsibilities are separated:

| Role | Does | Does not do |
|---|---|---|
| Data-ingestion engineer/service | Collects permitted documents and runs extraction | Approve clinical truth |
| Optional extraction LLM | Suggests structured text, table rows and links to source locations | Publish or approve content |
| Data QA reviewer/service | Finds broken structure, missing units, duplicate rows and bad page links | Decide clinical meaning alone |
| Clinical pharmacologist or qualified clinical reviewer | Approves medicine composition, safety meaning, population/form scope and exceptions | Approve legal rights without legal review |
| Regulatory pharmacist/legal reviewer | Reviews regulatory events, legal interpretation and source permissions where needed | Approve unrelated clinical content |
| Knowledge owner | Records the approval and publishes the reviewed version | Bypass required clinical or legal approval |
| Platform/retrieval engineer | Builds indexes, runs retrieval tests and records versions | Change approved clinical content silently |
| Doctor using BRAHMO | Reviews the final answer during a consultation | Manually approve every source chunk |

The doctor does not approve every chunk. Qualified reviewers approve the knowledge before it serves. The doctor reviews the answer generated from that approved knowledge and remains responsible for the final clinical decision.

Each chunk stores:

```text
source ID and version
document title and section heading
page or stable location
question/decision type
condition and specialty
population and formulation
jurisdiction and effective date
recommendation or fact
exceptions and contraindications
units and table relationships
footnote references
supersession and rights status
```

Headings are retained. Tables are stored both in readable form and in structured cells. Units are preserved exactly and normalised only as an additional field. A chunk is not published if extraction loses an exception, footnote or cross-page dependency.

Compact example:

```text
Chunk ID: ICMR-STW-2022-HTN-0042
Source: ICMR Standard Treatment Workflow, Volume 1, 2019 edition
Location: Hypertension in Adults, section â€œFollow-upâ€, page 7
Scope: adult; hypertension; India; reference only
Claim: [reviewed recommendation stored here]
Exception: [linked exception or population limitation]
Citation: source ID + version + page + heading
```

The example shows the structure; the clinical claim must be copied only after source review. A citation validator checks that the cited location contains the claim. An applicability validator separately checks population, formulation, date and jurisdiction.

### 5. Retrieval and embedding/index strategy

Use three complementary indexes:

1. An exact index for medicine names, ingredients, notification IDs, numbers and defined terms.
2. A metadata index for specialty, condition, population, formulation, country, source status, version and effective date.
3. A semantic embedding index for paraphrased questions and related wording. Similarity thresholds and reranking weights are calibrated on evaluation sets and then checked on independent held-out cases; they are not chosen by intuition alone.

The query first applies security and scope filters, then runs exact and semantic search. Results are reranked by applicability, authority, current status and evidence completeness. Similarity thresholds are tuned from evaluation results, but retrieval similarity is still not proof of clinical applicability or truth.

### 5a. Retrieval sequence and safe fallback

The serving sequence is:

```text
Exact search + semantic search
                |
                v
Reciprocal Rank Fusion (RRF)
combines the ranked lists
                |
                v
Candidate list is evaluated for scope
                |
                v
Validated reranker orders the candidates
                |
        +-------+-------+
        |               |
 Reranker available  Reranker unavailable
        |               |
        v               v
Use reranked list   Use RRF list as a
                    visible degraded fallback
                |
                v
Take the top candidate evidence set
                |
                v
Run evidence-sufficiency and applicability checks
                |
        +-------+-------+
        |               |
      Pass             Fail
        |               |
        v               v
Send only the       Abstain, ask for
validated evidence  clarification or show conflict
to the LLM
```

RRF is a useful baseline because it combines independent exact and semantic rankings without requiring them to use the same score scale. If the reranker fails, the RRF list may be used only as a degraded fallback; the answer must show the degraded state and apply a stricter evidence gate.

The system should not blindly send twenty chunks to the LLM. The candidate pool size and final evidence count are tuned on evaluation sets. A practical starting design is to retrieve a larger candidate pool, rerank it, then send only the small set of non-duplicated, applicable evidence units needed for the answer. The LLM receives evidence, not an unfiltered search dump.

### 5b. Multi-stage retrieval evaluation

Evaluation must show where quality is lost. The early stages should favour recall; the later stages should favour precision, applicability and citation support.

```text
Gold evidence set for a question
        |
        v
Measure exact + semantic retrieval recall
        |
        v
Measure RRF recall
        |
        v
Measure reranker recall and ranking quality
        |
        v
Measure applicability filtering
        |
        v
Measure evidence-sufficiency decisions
        |
        v
Measure citation support and final answer faithfulness
```

For each stage, record the numerator, denominator and failure reason. Useful measures include recall at the candidate pool size, recall at the final evidence size, precision of applicable evidence, the percentage of questions with sufficient evidence, citation-support rate and confidently wrong answer rate. If a gold passage is found by search but removed by the applicability filter, that is a filtering error; if it is never retrieved, it is a retrieval error.

The retrieval pipeline is therefore tuned in two directions:

- **Early retrieval:** maximise recall so that useful evidence is not lost.
- **Reranking and final selection:** improve precision, remove duplicates and exclude out-of-scope evidence.
- **Answer generation:** use only the validated final evidence set and abstain when it is insufficient.

All threshold changes are tested on development data and then confirmed on independent held-out cases. Optimising one stage must not hide a drop in citation support or an increase in confidently wrong answers.

The embedding model and index version are recorded. Re-embedding is a controlled data change and triggers retrieval and citation regression tests. Embedding text must respect source rights and must not include unnecessary patient data.

### 6. Conflict, correction and withdrawal handling

```text
New edition or regulatory event arrives
        |
        v
Compare identifiers, dates and affected scope
        |
        v
Mark old content as current, superseded, withdrawn or disputed
        |
        v
Re-run affected retrieval and answer tests
        |
        v
Publish only after required review
        |
        v
Keep old versions for audit, but prevent them serving as current evidence
```

If a canonical drug record contradicts another trusted record or a reviewed market record, it enters a quarantine queue. It cannot silently become truth. A local stock record cannot override the canonical identity or regulatory event store.

### 7. Missing rights and coverage consequences

If a source is accessible but its permission for storage, extraction, indexing or displayed excerpts is unclear, it is not placed in the serving corpus until the permission is resolved. The consequence may be:

- The affected question class is labelled unsupported.
- Only metadata or a link is retained, if permitted.
- The system abstains rather than reproducing the source.
- A separately licensed or curated source is required.

The source register therefore contains a rights status for every source and capability. â€œAvailable onlineâ€ is never recorded as â€œapproved for use.â€

### 8. Source quality gates

Before a source can serve consequential answers, the owner must provide:

- Identity, authority and intended purpose
- Version, publication date and effective date
- Geographic, population and formulation scope
- Rights decision for storage, extraction, indexing and display
- Extraction quality result and reviewer identity
- Conflict and supersession status
- Retrieval eligibility and withdrawal method

The final answer cites the source actually used, not merely the source family name.

## S4 Â· Rules of Behavior for India

### 1. Product boundary

BRAHMO is a clinician-facing decision-support and reference tool. It drafts, retrieves, maps and checks. The registered medical practitioner remains responsible for the clinical decision. The product must not be described as an autonomous prescriber, diagnostician or patient-advice service.

The legal statements below distinguish a source from my product interpretation. Final classification and launch approval require qualified Indian legal and regulatory review for the exact intended use, deployment and claims.

### 2. Enforceable behavior rules

| Condition | Required or prohibited behavior | Evidence/reason | Enforcement | Verification/approval |
|---|---|---|---|---|
| A consequential clinical claim is requested | Show a versioned source and claim-level location, or state that the claim cannot be confirmed | Fixed grounding requirement; unsupported generated clinical content is unsafe | Evidence-sufficiency gate blocks unsupported generation | Citation-faithfulness and abstention tests; clinical reviewer approval |
| Medicine identity is ambiguous | Ask the doctor to clarify; do not silently choose a fuzzy match | A wrong ingredient invalidates safety checks | Resolution service returns `UNVERIFIED_INPUT` and blocks the safety conclusion | Held-out brand, spelling and transliteration mapping set; zero silent wrong mappings |
| A medication-safety check lacks coverage or input | Display the exact non-safe state; never display â€œsafeâ€ | Missing data is not evidence of no risk | Rule engine returns the eight-state contract | Engineering tests every state and clinical review of wording |
| A fixed-dose combination is entered | Decompose it into ingredients before checking | Indian prescriptions may hide duplicate ingredients inside combinations | Drug master requires reviewed ingredient composition | Drug-data review and FDC regression tests |
| Regulatory status is requested | Use effective-dated regulatory events with notification, dates, scope and supersession; do not use a permanent banned boolean | A product can be in stock while regulatory status differs | Regulatory event store and conflict queue | Regulatory reviewer verifies event records and release eligibility |
| A local clinic or hospital protocol applies | Label it as tenant-scoped and show its owner, date and scope | Local practice may be useful but is not national truth | Tenant filter prevents cross-organization serving | Cross-tenant tests and organization approval |
| Patient data is processed | Use only the approved purpose, access role and organization scope; minimize logs | The [Digital Personal Data Protection Act, 2023](https://www.indiacode.nic.in/indiacode/handle/123456789/22037?view_type=browse) establishes a statutory framework for processing digital personal data; applicability, commencement and operational obligations must be reviewed for deployment | Authorization, encryption, retention, deletion and incident procedures | Privacy/legal review and negative access tests |
| The software's intended use may meet a medical-device definition | Do not claim â€œnot a medical deviceâ€ based only on a disclaimer; pause affected features until classification is reviewed | CDSCO states that software can fall within medical-device regulation depending on intended use and risk; see [CDSCO medical-device information](https://www.cdsco.gov.in/opencms/opencms/en/Medical-Device-Diagnostics/Medical-Device-Diagnostics/) and [software classification notice](https://cdsco.gov.in/opencms/resources/UploadCDSCOWeb/2018/UploadPublic_NoticesFiles/Software.pdf) | Feature-classification register and launch blocker for unresolved high-risk use | Indian regulatory counsel and responsible product owner approval |
| A telemedicine workflow is enabled | Keep the clinician/AI boundary explicit and require the registered practitioner to make the clinical decision | The [Telemedicine Practice Guidelines dated 25 March 2020](https://www.mohfw.gov.in/pdf/Telemedicine.pdf) apply to registered medical practitioners using telemedicine; whether a particular BRAHMO workflow is telemedicine must be reviewed | Feature flags and workflow classification prevent unsupported remote-prescribing behavior | Legal and clinical review of the exact workflow |
| A source is publicly accessible | Do not assume that access permits storage, extraction, indexing, displayed excerpts or model training | Copyright and contract permissions are separate from technical access | Rights register blocks unapproved source ingestion | Source-owner/legal approval and rights audit |
| Patient-facing advice is requested | Do not expose the clinician tool as a patient advice surface | The fixed assessment fence excludes patient-facing AI advice | Product routing refuses or redirects the request | UI tests and release review |
| A document contains instructions to the AI | Treat the text as data, not as a system instruction | Documents and transcripts are untrusted input | Prompt isolation and tool authorization | Prompt-injection test corpus and security approval |
| A model, prompt, source, rule or mapping changes | Do not silently promote it | Fixed no-silent-upgrade requirement | Release manifest, mapped regression and rollback | Engineering, QA and relevant clinical/legal approval |

### 3. Legal and regulatory source register

| Area | Primary source and status | Technical consequence | Open point and reviewer |
|---|---|---|---|
| Digital personal data | [Digital Personal Data Protection Act, 2023](https://www.indiacode.nic.in/indiacode/handle/123456789/22037?view_type=browse), enacted 11 August 2023. The Act provides for commencement by government notification and different provisions may commence on different dates. | Maintain purpose, access, retention, correction/deletion and incident controls as a product requirement; do not claim complete compliance from a privacy policy alone. | Confirm the provisions and rules in force for the deployment date, roles of data fiduciary/processor, children and cross-border posture. Reviewer: Indian privacy counsel. |
| Medical-purpose software | [Medical Devices Rules, 2017](https://cdsco.gov.in/opencms/opencms/en/Acts-and-rules/Medical-Devices-Rules/) and CDSCO's software classification material. | Intended use, risk and claims drive classification work; maintain quality, safety, performance and post-market obligations if applicable. | Obtain a written feature-by-feature classification opinion before enabling high-risk claims. Reviewer: Indian medical-device regulatory counsel. |
| Telemedicine | [Telemedicine Practice Guidelines, 25 March 2020](https://www.mohfw.gov.in/pdf/Telemedicine.pdf). | If the workflow is telemedicine, preserve registered-practitioner responsibility and applicable consultation/prescribing constraints. | Confirm whether BRAHMO's exact doctor-facing workflow is telemedicine or decision support used within another consultation. Reviewer: health-law counsel and clinical lead. |
| Professional conduct | [NMC rules and regulations page](https://nmc.org.in/page/rules-regulations-rules-regulations-nmc), including the listed Registered Medical Practitioner (Professional Conduct) Regulations, 2023 and amendment/abeyance notices. | Do not assert that AI approval transfers accountability from the practitioner. Product copy must identify the doctor as decision-maker. | Confirm the current operative professional-conduct instrument and any prescription/generic-name requirement relevant to the shipped workflow. Reviewer: medical-law counsel and RMP reviewer. |
| Drugs and restrictions | [CDSCO Drugs Rules](https://cdsco.gov.in/opencms/en/Acts-and-rules/Drugs-Rules/) and [CDSCO notifications](https://cdsco.gov.in/opencms/opencms/en/Notifications/Gazette-Notifications/). | Use notification-level, effective-dated events and scope; do not infer legality from stock or a brand label. | Confirm which schedules, restrictions and reporting duties are in scope for each enabled capability. Reviewer: regulatory pharmacist/legal counsel. |

### 4. What the product may and may not answer

The S4 behavior flow is:

```text
Doctor requests a feature or answer
                |
                v
Identify the request type
clinical reference / medicine check /
regulatory question / patient data
                |
                v
Is it inside the fixed product boundary?
                |
        +-------+-------+
        |               |
       Yes              No
        |               |
        v               v
Continue          Refuse or abstain
        |
        v
Check required source, data and permissions
        |
        v
Are the source, rights and legal position clear?
        |
      +---+---+
      |       |
     Yes      No
      |       |
      v       v
Continue   Narrow, disable
           or mark unverified
      |
      v
Run deterministic safety and access controls
      |
      v
Generate only a grounded, labelled response
      |
      v
Doctor reviews and decides
      |
      v
Save approval, rejection, sources, versions
and the audit record
```

The product may provide a cited reference answer, summarize an approved source, identify a medicine when the mapping is unambiguous, show ingredient composition, display a deterministic check result and identify what is missing.

The product must refuse or abstain from patient-specific dose calculation, diagnostic ranking, triage/severity scoring, auto-generated treatment plans and patient-facing advice because those are outside the fixed assessment fences. It must also abstain when evidence is not applicable, when a medicine is ambiguous, or when a required safety service is unavailable.

### 5. Approval and unresolved matters

The following cannot be marked ready merely because the prototype works:

- Applicable DPDP commencement, rules and deployment obligations
- Medical-device classification for the exact intended use and claims
- Whether any enabled workflow is telemedicine
- Operative professional-conduct and prescription requirements
- Rights to store, extract, index and display each source
- Coverage and sign-off for every safety rule and regulatory event

Until the responsible reviewer records an approval or a documented limitation, the affected capability remains disabled, narrowed or explicitly labelled unverified. A disclaimer cannot replace a required approval.

## S5 Â· Correctness & Safety Design

### 1. Deterministic and generated behavior

The system has two different behavior types. They must not be confused.

| Behavior | Responsible component | Can it use an LLM? | Required control |
|---|---|---:|---|
| Brand and alias resolution | Drug master and resolution service | No for final truth | Unique match, scope and confidence checks; ambiguity goes to the doctor |
| Ingredient and FDC decomposition | Versioned drug data | No | Structured records and reviewed mappings |
| Duplicate and interaction checks | Safety rule engine | No | Pure rules over versioned inputs and data |
| Regulatory status | Regulatory event store | No | Effective-dated events, scope and supersession |
| Evidence retrieval | Retrieval service | No for source truth | Exact, metadata and semantic search followed by applicability checks |
| Question interpretation | Orchestrator/LLM | Yes, constrained | Structured output schema and regression tests |
| Answer wording and summarisation | LLM | Yes, constrained | Retrieved evidence only, citations and abstention |
| Final clinical decision | Doctor | No automation | Doctor review and explicit confirmation |

The LLM is a composition tool. It is not the authority for medicine identity, safety findings, regulatory status or patient-specific treatment decisions.

### 2. Required result for a medication check

Every medication finding must include:

```text
medicine as entered
resolved ingredient/salt
strength and dose form
combination ingredients, if applicable
patient-context fields used
rule identifier and rule version
data source and effective date
scope: population, formulation and jurisdiction
finding status
plain-language explanation
```

The system must show the difference between:

- `HIT`: a configured rule found a relevant issue.
- `CHECKED_NO_HIT`: the available, applicable rule coverage was checked and found no issue.
- `PARTIAL_COVERAGE`: only part of the requested check was covered.
- `UNVERIFIED_INPUT`: a required input could not be trusted or identified.
- `NOT_CHECKED`: the check did not run or was outside the enabled capability.
- `DATA_EXPIRED`: required data is outside its approved freshness period.
- `SOURCE_CONFLICT`: applicable sources disagree.
- `SERVICE_UNAVAILABLE`: a required service was unavailable.

Only `CHECKED_NO_HIT` may be presented as a completed check with no finding. The other states must remain visible to the doctor.

### 3. Missing, conflicting and stale information

```text
Required information missing
        |
        v
Identify exactly what is missing
        |
        v
Ask the doctor for clarification when possible
        |
        v
If clarification is not provided
        |
        v
Return an explicit incomplete or unverified status
```

If two applicable sources disagree, the system does not merge them into one answer. It displays both sources, dates, scope and the nature of the disagreement. A qualified reviewer receives the conflict for resolution.

If a source or rule is superseded, the newer approved version serves and the older version remains available for audit. If the system cannot establish which version applies, it abstains from the consequential claim.

### 3a. Graceful handling of uncertainty

When BRAHMO cannot confirm an answer, it must do two things:

1. Tell the doctor clearly what is missing, conflicting or unavailable.
2. Route the case to the correct queue when follow-up is needed.

```text
System cannot confirm answer
        |
        v
Explain the reason to the doctor
        |
        v
Classify the failure
        |
   +----+-------------+----------------+
   |                  |                |
Service failure   Missing/ambiguous   Clinical conflict
   |                  |                |
   v                  v                v
Retry or           Clarification      Qualified clinical
technical DLQ     queue              review queue
```

The queues have different purposes:

| Queue | Use | Required handling |
|---|---|---|
| Technical dead-letter queue | Timeout, malformed payload, failed extraction or unavailable dependency | Retry safely, alert engineering and keep the case visible as unavailable |
| Clarification queue | Missing strength, formulation, patient context or ambiguous brand | Ask the doctor for the specific missing value; do not guess |
| Clinical review queue | Conflicting sources, incomplete high-risk coverage or a possible new safety issue | Send an authorized structured case bundle to a qualified clinical reviewer |
| Security/incident queue | Prompt injection, unauthorized access or suspected data leakage | Restrict access, preserve evidence and follow the security incident process |

The human-review bundle contains only the minimum authorized information needed:

```text
original question
relevant patient-context version
medicine input and resolution result
retrieved source IDs and versions
missing or conflicting information
checks already performed
service status and failure reason
audit ID and tenant
```

The doctor sees a graceful response such as:

```text
I cannot confirm this from the currently verified sources.
Missing information: the medicine formulation is unclear.
Action: please confirm tablet, capsule or liquid form.
No safety conclusion has been produced.
```

For a source conflict:

```text
I cannot merge these sources because they disagree.
Source A: [version/date/scope]
Source B: [version/date/scope]
Action: the conflict has been sent for qualified review.
No unsupported conclusion has been generated.
```

The system must not invent a missing value, select a winner silently, present an unknown result as safe or allow the LLM to hide the reason for abstention. The case may later become a de-identified regression or gold evaluation case only after qualified review establishes the expected behavior.

### 4. Patient-context corrections

The doctor may correct a patient-context field, such as an allergy or current medicine. The correction is treated as a consultation event, not automatically as permanent patient truth.

```text
Doctor corrects a field
        |
        v
Show the changed value and source
        |
        v
Use it for the current review if authorized
        |
        v
Record who changed it and when
        |
        v
Send for patient-record or clinical review where required
        |
        v
Promote to a durable record only through the approved workflow
```

The original value is retained for audit. A correction cannot silently change another consultation or another tenant's record.

### 5. Source corrections and rule changes

Source or rule corrections follow the same controlled lifecycle:

```text
Correction received
        |
        v
Capture reason, source and affected records
        |
        v
Assess immediate safety impact
        |
        v
Disable or withdraw affected content if necessary
        |
        v
Qualified reviewer approves the corrected version
        |
        v
Run mapped regression and replay tests
        |
        v
Publish a new version and release manifest
```

Past outputs remain tied to the versions that produced them. A correction does not rewrite history.

### 6. Doctor-facing safety behavior

The interface must make the following visible:

- Whether the medicine identity is confirmed or ambiguous
- Which checks ran
- Which checks were incomplete
- Whether evidence is Indian, organization-specific or international
- Source version and date
- Conflicting or superseded sources
- Whether a response is a reference answer rather than a patient-specific treatment plan
- What the doctor must clarify or review

High-severity findings require an explicit acknowledgement or review action before the doctor can complete the relevant workflow. The system must not use color alone; the text and status must also be clear.

### 7. Access control and multi-tenancy

Every request is authorized against:

```text
user â†’ organization â†’ clinic â†’ patient â†’ consultation â†’ capability
```

An organization protocol can affect only that organization and is visibly labelled. It cannot override a global safety or regulatory block. Shared-device sessions require user identification, short inactivity expiry and re-authentication before viewing sensitive records.

All authorization failures are denied by default and recorded without placing unnecessary patient information in logs.

### 8. Language and degraded operation

The system may accept mixed English and Indian-language input, but recognition or translation uncertainty is preserved. A medicine name that changes during translation or transliteration must be re-confirmed before checking.

When a service is unavailable:

```text
Dependency fails
        |
        v
Identify affected capability
        |
        v
Show a visible degraded-state message
        |
        v
Return the correct non-safe status
        |
        v
Do not silently use stale or unrelated content
        |
        v
Record failure and recovery in the audit trail
```

Cached reference content may be shown only with its version and freshness. Cached content cannot silently substitute for a current regulatory event or a current medication-safety service.

### 9. Prompt-injection and untrusted documents

Transcripts, uploaded documents and retrieved passages are treated as data. Instructions inside them cannot change system rules, reveal secrets or authorize actions. The orchestration layer separates system instructions, tool results, source content and doctor input. Prompt-injection cases are included in release testing.

### 10. Correctness boundary

The product is correct only when the complete chain is valid:

```text
Correct input
â†’ correct medicine identity
â†’ applicable versioned data
â†’ correct deterministic checks
â†’ sufficient evidence
â†’ faithful cited draft
â†’ doctor review
â†’ complete audit
```

A strong language-model answer cannot compensate for a wrong medicine mapping, missing patient context, stale rules, an unavailable service or an absent citation. Any broken link must be visible and must reduce the system's claim rather than being hidden.

## S6 Â· What You Added

The supplied architecture has a strong safety foundation, especially its doctor-approval boundary, deterministic medication checks, versioning and abstention behavior. The following additions make the design more operationally complete and reduce ambiguity at the points where a prototype could otherwise be mistaken for doctor-ready national coverage.

| Factor | Current omission or weak assumption | Consequence | Severity | Proposed handling |
|---|---|---|---|---|
| LLM-assisted knowledge extraction | The raw â†’ structured â†’ verified â†’ live lifecycle does not define whether an LLM may extract facts from difficult documents. | An extraction error could silently become live clinical knowledge. | Critical | Add an offline extraction lane with provenance, schema checks, clinician review and a hard barrier preventing unverified records from serving. See S2 D11. |
| Validated coverage versus national availability | The plan describes scale-out to all India, but availability is not the same as validated coverage across specialties, languages, medicines and populations. | Doctors may assume the system supports questions outside its tested scope. | High | Publish capability and coverage metadata; show supported populations, specialties, languages and source dates; abstain outside validated coverage. |
| Applicability separate from retrieval relevance | A semantically similar passage may concern a different population, formulation, jurisdiction or superseded edition. | The system may cite a relevant-looking but clinically inapplicable source. | Critical | Add metadata filters and a deterministic evidence-sufficiency/applicability gate. Test retrieval recall and applicability separately. See S2 D4. |
| Correction governance | A one-tap doctor correction is described, but a correction must not automatically become clinical truth. | Incorrect or context-specific corrections could contaminate the corpus or drug master. | High | Store corrections as tenant- and case-scoped events, route them to review, preserve the original value and promote changes only through the controlled data lifecycle. |
| Alert fatigue governance | The plan mentions acted-versus-dismissed telemetry but does not define alert classes, suppression rules or clinical review criteria. | Doctors may ignore high-severity findings after repeated low-value alerts. | High | Use severity tiers, deduplication and reason codes; review acted, dismissed and overridden findings; never suppress a critical alert solely because it is frequently dismissed. |
| Multilingual and transliteration safety | The context identifies mixed-language consultations and regional names, but language coverage and failure behavior are not fully specified. | A transliteration or speech-recognition error could produce the wrong medicine identity or wrong question. | Critical | Treat language recognition as uncertain input, preserve the original text/audio reference, test regional aliases and require clarification before consequential mapping. |
| Offline and shared-device security | Intermittent connectivity and shared computers are identified, but cached data, session expiry and local exposure are not fully defined. | Stale evidence may look current or one doctor's patient context may be exposed to another user. | Critical | Use explicit degraded banners, scoped read-only caches, short sessions, re-authentication, local data minimization and tenant/role negative tests. See S2 D7 and D9. |
| Regulatory and intended-use classification | The build stages expect a regulatory position to be documented, but a disclaimer alone is not a classification decision. | Features may be enabled before the applicable legal or medical-device review is complete. | Critical | Maintain a feature-by-feature legal classification record. Block affected capabilities until qualified review records the applicable status, scope and technical controls. |
| Source rights and permitted use | The current plan refers to rights-cleared sources but does not establish separate permissions for storage, extraction, indexing, excerpt display and model training. | A source may be accessible but not lawfully reusable in the proposed pipeline. | Critical | Track rights as structured metadata for every source and capability. Do not ingest or display material until the required permission is documented; restrict or remove affected capabilities when rights lapse. |
| Review-capacity scaling | The human lane correctly says scope should shrink when review backs up, but the operational trigger is not defined. | Unreviewed content may be allowed into production to maintain breadth. | High | Track review queue age, unresolved critical findings and approval status. Reduce supported scope before serving unreviewed safety content. |
| Audit completeness and tamper evidence | Replayability is required, but the minimum event set and protection against incomplete or altered logs are not specified. | An incident investigator may be unable to reconstruct what the doctor saw and what data produced it. | High | Record input, context version, evidence IDs, rule/data/model/prompt versions, outputs, doctor action, service states and timestamps; test replay and unauthorized modification. See S2 D8. |

These additions do not replace the fixed safety laws. They make the existing architecture enforceable by defining boundaries, approval states, degraded behavior, scope and evidence needed for release. Each item is linked to a design decision in S2 and should be tested through the validation plan in S7.

## S7 Â· Validation & QA Plan

### 1. Validation principle

BRAHMO must not rely on one overall accuracy score. A high average score cannot hide one unsafe medicine mapping, one cross-tenant leak or one unsupported consequential claim. Catastrophic failures are release blockers even when the average result looks good.

Engineering and QA verify whether the system behaves as designed. Qualified clinical and legal reviewers approve whether the content and intended use are acceptable. These responsibilities are separate.

### 2. Evaluation-set construction

Every evaluation case has a stable ID, source/provenance, scope, expected behavior and owner. The set contains:

- Realistic but de-identified doctor questions and prescription patterns
- Brand names, misspellings, transliterations and look-alike names
- Single-ingredient and fixed-dose combinations
- Adult, pediatric, pregnancy/lactation and elderly contexts where supported
- Different strengths, formulations and incomplete patient contexts
- Indian clinical, programme and regulatory questions
- Conflicting, superseded, stale and unavailable sources
- Mixed-language input and speech/transcription errors
- Prompt-injection documents and malicious imported text
- Tenant isolation, shared-device and permission cases
- Offline, timeout and partial-service failures

Clinical reference answers are created or approved by qualified reviewers using cited source versions. Drug-mapping and rule keys are held by an evaluator independent of the implementation author. Test cases are split into development, regression and independent held-out sets. Held-out cases are not edited merely to make a release pass.

### 3. Capability test matrix

| Capability | Test data/reference standard | Method | Pass criterion | Evidence | Validator |
|---|---|---|---|---|---|
| Brand-to-ingredient mapping | Independent reviewed mapping key with regional aliases and variants | Exact, fuzzy, transliteration and formulation tests | Zero silent wrong mappings; ambiguous cases return clarification | Case-level results and error taxonomy | Data QA lead and clinical pharmacologist |
| FDC decomposition | Reviewed ingredient-composition records | Compare output ingredients, strengths and forms | No missed or invented ingredient; source and version attached | Mapping diff and review record | Clinical pharmacologist |
| Regulatory status | Versioned CDSCO event records | Replay events by effective date and scope | Correct current/past status; no permanent boolean substitution | Event replay report | Regulatory pharmacist/legal reviewer |
| Medication safety | Clinician-approved rule cases, including incomplete inputs | Run deterministic engine with fault injection | Correct finding and exact status; missing data never becomes safe | Rule execution trace | Clinical pharmacologist and QA |
| Retrieval relevance | Gold evidence passages and hard negatives | Exact, metadata and semantic retrieval tests | Required evidence is retrieved within approved rank/recall threshold; irrelevant scope is filtered | Retrieval report | Retrieval engineer and clinical reviewer |
| Applicability | Cases tagged by population, form, jurisdiction and date | Separate relevance from applicability scoring | No out-of-scope evidence used for a consequential claim | Applicability decisions and examples | Clinical reviewer |
| Citation faithfulness | Reviewer-approved claim-to-source links | Check every consequential claim and location | Every consequential claim is supported or abstained; source location resolves | Citation audit | Clinical reviewer and QA |
| Abstention | Cases with missing, conflicting, stale or unavailable evidence | Remove or corrupt dependencies | Correct refusal/status and clear missing-information explanation | Abstention report | QA and clinical reviewer |
| LLM composition | Grounded prompts and adversarial unsupported questions | Compare draft with retrieved evidence | No unsupported consequential claims; required labels and citations present | Prompt/model regression report | QA lead and clinical reviewer |
| Prompt injection | Documents containing instructions or malicious content | Ingest and ask controlled questions | Document text cannot alter system policy, tools or data access | Security test evidence | Security reviewer |
| Tenant isolation | Cross-tenant patient, protocol and audit cases | Negative authorization and search tests | Zero unauthorized reads, writes or evidence leakage | Access-control report | Security lead |
| Audit replay | Golden-thread cases with pinned versions | Reconstruct displayed result | Inputs, evidence, rules, model, prompt and doctor action are reconstructable | Replay artifact | Platform QA |
| Multilingual input | Clinician-reviewed language and transliteration cases | Speech/text recognition and mapping tests | Uncertain recognition is shown; no silent consequential remapping | Language error report | Language reviewer and clinical reviewer |
| Degraded operation | Injected network, data and service failures | Run each dependency failure independently | Visible degraded state; correct non-safe status; no quiet fallback | Failure-injection report | SRE/QA lead |

### 4. Metrics and denominators

Metrics are reported with their numerator, denominator, scope and confidence interval where useful. Examples:

```text
Silent wrong mapping rate
= wrong medicine mappings not flagged as uncertain
  / all evaluated medicine mappings

Confidently wrong answer rate
= consequential answers judged materially wrong and presented without
   an appropriate warning or abstention
  / all consequential answered cases

Supported-answer rate
= answers judged useful and supported by applicable evidence
  / all cases where the system produced an answer

Appropriate-abstention rate
= cases where the system abstained when evidence was insufficient,
   conflicting, stale or unavailable
  / all cases requiring abstention

Citation support rate
= consequential claims with a valid source and location
  / all consequential claims shown

Cross-tenant leakage rate
= unauthorized data exposures
  / all authorization test attempts
```

The target for silent wrong mappings, cross-tenant leakage and an unknown status rendered as safe is zero for the tested release scope. Thresholds for retrieval and useful supported answers are set per capability and reviewed by the responsible clinical owner; they are not replaced by one weighted average.

### 5. Release gates

Every release manifest lists changed code, model, prompt, source, rule, mapping, embedding and configuration versions. A change triggers the regression suite mapped to that change.

Release is blocked when any of the following occurs:

- A silent wrong medicine mapping is found.
- A missing or partial check is rendered as safe.
- A consequential claim lacks valid support.
- A cross-tenant or unauthorized access test fails.
- A prompt-injection test changes policy or reveals protected data.
- An audit replay cannot reconstruct a consequential output.
- A required clinical, legal or source-rights approval is missing.
- A newly changed rule, model, prompt or source has not completed its mapped evaluation.

Passing a release gate proves only the tested scope. It does not prove national coverage.

### 6. QA operating model

Daily QA duties include reviewing new failures, checking ingestion and rule changes, confirming service-status rendering and triaging high-severity incidents. Weekly QA duties include trend review, sampling production traces, reviewing newly added source content, checking alert actions/dismissals and refreshing the risk-based evaluation queue.

Production sampling is stratified rather than purely random. It oversamples new capabilities, high-severity findings, abstentions, corrections, ambiguous mappings, languages and tenants with unusual error patterns. Patient data is minimized or de-identified for QA review.

### 7. Production failures become regression cases

When a production query fails, the team records the failure and its cause:

```text
Production failure
        |
        v
Protect and de-identify the case
        |
        v
Triage: retrieval, mapping, rule, model,
source, access or interface failure
        |
        v
Qualified reviewer defines the expected behavior
        |
        v
Add the case to the regression/evaluation set
        |
        v
Add it to the gold set only after approved reference evidence exists
```

The case is not automatically treated as gold truth. If it is used to tune a model, prompt, threshold or rule, it remains excluded from the independent held-out set. This prevents production feedback from leaking into the final measurement while ensuring that repeated failures are tested in future releases.

The small-team model remains workable by using automation for schema checks, replay, citation validation, authorization tests, prompt-injection tests and sampling selection. Qualified reviewers focus on high-risk and novel cases. If review queues or unresolved critical findings exceed the approved limit, the supported capability shrinks before unreviewed content is released.

### 7. Revalidation triggers

Re-run the mapped evaluation and obtain the required approval after:

- A model or model-provider change
- A prompt or orchestration change
- A source edition, regulatory event or source withdrawal
- A drug mapping or safety-rule change
- An embedding model or index rebuild
- A translation, speech or language-pipeline change
- A patient-context or Scribe contract change
- A security, privacy or authorization change
- A serious incident, newly discovered failure or repeated doctor correction pattern

Production quality is monitored after release, and rollback is available if a critical regression appears.

## S8 Â· Doctor-Handover Checklist

The product is ready for handover only for the capability and coverage that have actually been validated. National availability must not be described as national clinical coverage.

| Item | Required result | Verification/evidence | Responsible validator/approver | Handover blocker |
|---|---|---|---|---|
| Intended use | The shipped capability has a written, approved description | Versioned intended-use statement and feature list | Product owner and legal reviewer | Yes, if missing or inconsistent with the UI |
| Fixed safety boundaries | No patient-specific dose calculation, diagnosis ranking, triage scoring, auto-treatment plan or patient-facing advice | Scope tests and UI review | Clinical lead and QA | Yes |
| Clinical content | Every serving clinical source is verified, current, scoped and cited | Source register, review records and citation tests | Clinical reviewer | Yes for affected capability |
| Drug identity | Supported brands resolve correctly to ingredient, strength and form | Held-out mapping report with zero silent wrong mappings | Clinical pharmacologist and data QA | Yes |
| FDC handling | Combination products are decomposed before checks | FDC mapping and regression evidence | Clinical pharmacologist | Yes |
| Safety rules | Critical rule set is approved and deterministic | Rule versions, test cases and signed review batches | Clinical pharmacologist | Yes for affected rule |
| Unknown states | All eight states render correctly and missing data never appears safe | UI contract tests and screenshots/logs | QA lead and clinical reviewer | Yes |
| Regulatory data | Events contain source, dates, scope and supersession | Regulatory event register and review record | Regulatory pharmacist/legal reviewer | Yes for regulatory features |
| Patient context | Scribe and patient-context contracts are versioned and validated | Contract tests, sample payloads and failure tests | Integration owner and clinical lead | Yes |
| Tenant isolation | No patient, protocol or audit data crosses organization or role boundaries | Negative authorization and cross-tenant test report | Security lead | Yes |
| Privacy controls | Access, retention, deletion, logging and incident processes are configured | Privacy review, configuration evidence and runbook | Privacy/legal reviewer and security lead | Yes for live patient data |
| Source permissions | Storage, extraction, indexing and display rights are recorded | Rights register and contracts/permissions | Legal/source owner | Yes for affected source |
| Model and prompt versions | Serving model, prompts, tools and indexes are pinned | Release manifest and rollback package | Engineering owner and QA | Yes |
| State machine | Invalid transitions and unauthorized tools are denied | Transition tests and tool-allowlist report | Platform/security QA | Yes |
| Audit replay | A past result can be reconstructed from recorded versions | Golden-thread replay artifact | Platform QA and clinical reviewer | Yes |
| Degraded operation | Outages show visible degraded status and no quiet unsafe fallback | Failure-injection evidence | SRE/QA lead | Yes for affected dependency |
| Language coverage | Supported language and transliteration limits are visible | Language evaluation and known-limit list | Language reviewer and product owner | Yes if claims exceed evidence |
| Doctor instructions | Doctors know what the output means and when to clarify or ignore it | Short user guide, examples and training acknowledgement | Clinical product owner | Yes |
| Support and incidents | A named route exists for corrections, safety incidents and access issues | Runbook, escalation contacts and response process | Operations owner | Yes |
| Production monitoring | Quality, latency, failures, abstentions, corrections and alert actions are monitored | Dashboard definitions and sampling procedure | QA/SRE owner | Yes |
| Clinical sign-off | Qualified reviewers have approved the exact shipped scope | Signed release approval with capability list | Clinical approver | Yes |
| Legal/regulatory sign-off | Applicable classification and obligations are documented | Legal/regulatory memo for the shipped scope | Qualified legal/regulatory approver | Yes |

### Handover decision flow

```text
Capability is proposed for release
                |
                v
Is the exact scope documented?
                |
        +-------+-------+
        |               |
       Yes              No
        |               |
        v               v
Continue          Handover blocked
        |
        v
Are clinical, legal, source-rights,
security and engineering gates green?
        |
      +---+---+
      |       |
     Yes      No
      |       |
      v       v
Continue   Block, narrow or disable
      |
      v
Can the doctor understand the coverage,
uncertainty and required actions?
      |
    +---+---+
    |       |
   Yes      No
    |       |
    v       v
Handover   Improve instructions and retest
approved
```

The handover record names the approved capability, validated populations and specialties, supported languages, source versions, known limitations, blocked features, responsible owners and rollback method. A missing approval is not a completed checkbox.

## S9 Â· AI-Usage Note


1. I used AI to help explore and design the architecture, compare alternatives, structure the decision register, identify safety gaps, create flowcharts and organize the validation and handover plans. I reviewed the recommendations and remain responsible for the final decisions and factual claims.

2. I independently checked and retained the recommendation that medication-safety checks must run outside the LLM using deterministic, versioned data. I also added an explicit state machine and tool allowlist because the supplied architecture did not define how prompt injection or invalid workflow transitions would be prevented.

3. The most important remaining uncertainty is the exact Indian legal and regulatory classification of the shipped software, including medical-device, privacy and telemedicine obligations. The affected capability must not be treated as approved until qualified reviewers confirm the position for the actual deployment and intended use.
    

