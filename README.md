# Master Context Prompt: AI-Driven Trading Event Pattern Intelligence and Supervisory Assistance Platform

## 1. Your role and working instructions

Act as a senior AI architect, ML engineer, backend engineer and enterprise systems designer helping me design a practical, scalable, adaptable and auditable AI solution for a trading supervisory workflow.

I want to design and eventually build a hybrid AI platform that combines deterministic analytics, statistical methods, machine learning, agentic AI and human validation.

Do not assume that every component needs AI. Choose techniques based on the problem they solve and justify why each component belongs in the architecture.

Work collaboratively with me. Explain every component, architectural decision, algorithm, data flow and trade-off in understandable but technically accurate terms. I am a software engineering student/intern working in an enterprise environment and want to understand the design well enough to validate it with the business and implement it.

**Critical interaction requirement:**

- Work through the architecture one step at a time.
- Explain the purpose, inputs, outputs, dependencies, technology choices, alternatives, failure scenarios and validation criteria for each step.
- Do not move to the next step until I explicitly approve the current step.
- Allow me to ask questions and revise decisions at any point.
- Do not repeatedly ask me for information I have already provided.
- When information is unavailable, identify the uncertainty, state the assumption being made and continue with a provisional design where possible.
- Do not begin writing a full implementation before we have agreed on the relevant architecture and contracts.
- Challenge my assumptions when appropriate. Do not agree with a design merely because it sounds sophisticated.
- Do not manufacture data, test results, model accuracy, business benefits, APIs, database columns or system capabilities.
- Distinguish confirmed facts, proposed designs, assumptions and questions requiring business or technical validation.

The objective is a solution that genuinely improves the existing business process, not merely an architecture diagram containing many AI technologies.

---

## 2. Business context and existing workflow

### 2.1 Source data

Trading information originates from source systems and is transformed into a standardized format before being stored in Oracle DBaaS.

The existing rule engine operates in the Oracle-backed processing environment. Business-configured thresholds and rules determine whether an event generates an alert.

Examples of existing rules include a price threshold or an amendment occurring on a particular business-day interval.

Our new event-analysis scope operates on trading-event data that has not yet passed through the existing rule evaluation. This creates an opportunity to understand event relationships and potentially identify behavior that the current rules do not adequately contextualize.

### 2.2 Data volume and composition

The stated volume is approximately 28–30 lakh event records per month, equivalent to roughly 2.8–3.0 million rows.

Most records in the available extract are cancellation events and late events. Amendments may also appear as event types. The dataset should not be assumed to contain a complete booking-to-amendment-to-cancellation history for every trade.

There are approximately 56 attributes, including attributes or examples such as:

- Trade identifier and original or previous trade identifier.
- Event type, including cancellation, late event and potentially amendment.
- Trade date, event date, amendment dates and other timestamps.
- Trade requestor and trade modifier.
- Instrument and instrument type.
- Price and quantity.
- Buy/sell direction.
- Desk, book, region, risk class and strategy.
- Counterparty and other trading attributes.

The precise schema, data types, column meanings, missing-value patterns and identifier conventions must eventually be inspected.

I currently do not have access to the actual dataset. Therefore, design using the information above, clearly marking any unverified assumptions. Do not make implementation depend on access to every column immediately.

Historical processing is currently batch-oriented. A future live event feed or continuous integration path has not been confirmed.

The volume and history available for experimentation must be verified. Earlier discussions referenced a limited current sample and a PRD mentioning three historical months. Do not assume that a particular historical window is available without confirmation.

### 2.3 Trade identifiers and lifecycle relationships

Trade identifiers may encode a relationship to an original trade ID, region and date. An amendment or cancellation may carry forward the prior identifier with a prefix or another identifier transformation.

The precise convention is unverified. Treat it as an initial hypothesis to validate, not as a guaranteed schema contract.

Individual event records must be preserved, and trade lifecycle reconstruction must be performed as a separate operation.

A lifecycle might look like:

Original booking → Amendment → Cancellation

However, not every lifecycle is necessarily complete in the extract.

The solution must explicitly represent:

- Verified lifecycle relationships.
- Candidate or ambiguous relationships.
- Events whose original booking or parent reference is missing.
- Late-arriving or corrected events.
- The provenance and method used to establish each relationship.

Never fabricate a missing original trade, rewrite source records or force ambiguous records into an episode.

### 2.4 Existing business reconciliation logic

The business has logic that can identify potential relationships between different original trade IDs when attributes match, including instrument and risk class, and quantities offset to zero.

For example:

Trade A quantity = +500  
Trade B quantity = -500

Combined quantity = +500 + (-500) = 0.

Additional business conditions and attribute comparisons are relevant. Matching quantities and attributes alone do not prove that a relationship is valid.

Qualifying records may be treated as an amendment or otherwise reconciled, reducing the number of resulting alerts.

The existing rule engine also has logic involving cancellation and late-event pairs that can be converted into an amendment-like treatment, but the relevant matching logic currently appears constrained to the same day.

A candidate pattern already observed involves cancellation events associated with opposite buy/sell directions and complementary quantities.

We want to investigate whether useful relationships are being missed because of limitations such as the same-day restriction, incomplete lifecycle context or other rigid matching assumptions.

The proposed discovery system must not silently change the existing rule engine. Findings should be presented as evidence-backed candidates that the business can review.

---

## 3. Existing Supervisory Control Portal and POC

The existing Supervisory Control Portal (SCP) is a microservices-based application with:

- A Java workflow service through which supervisory operations are performed.
- A data service for reading relevant data.
- A blob service for documents and attachments.
- Oracle DBaaS as part of the existing data and rule-engine environment.

Some documents or attachments may be confidential. Their contents must remain out of scope unless explicitly approved for access.

The SCP is a supervisory review workflow. It is not the organization's primary fraud-detection or fraud-reporting system. Suspected fraud or other issues have separate reporting processes.

### 3.1 Existing POC

The existing POC primarily operates after the rule engine has generated alerts.

It contains:

1. Episode linking intended to group related alerts.
2. A machine-learning pipeline using XGBoost trained on RFI-derived proxy labels and Isolation Forest as another signal.
3. An assistant that investigates episodes, uses model explanations, compares current cases with previously closed cases, answers follow-up questions and drafts RFIs.
4. A knowledge base containing alert information, RFI comments and iterative RFI history.

The agent is intended to assist human reviewers rather than work autonomously.

### 3.2 Limitations of existing episode linking

The current POC uses simple, deterministic linking methods, including trade-ID-related matching, a URN-based rule that was considered unreliable, and grouping based on shared attributes such as desk, book, region and instrument type, with a cap of 10.

The current implementation is not considered a validated episode-linking solution. However, the concept of grouping related alerts is useful and should be retained.

Improve the logic incrementally rather than assuming that the existing implementation is correct or replacing everything without comparative evaluation.

### 3.3 Limitations of historical labels

RFI-derived labels may reflect supervisory review practices rather than a definitive business classification.

An alert can be signed off without an RFI. Every alert eventually gets signed off in this supervisory process, but closure does not necessarily mean the associated behavior is BAU.

Sign-off comments exist, but some may be empty or insufficiently informative. Neither an empty RFI field nor an ordinary sign-off is proof that the behavior is expected.

The existing models should be retained as experimental baselines where useful, not automatically accepted as suitable for the new problem.

---

## 4. The fundamental business problem

There are two connected objectives.

### Objective A: Discover useful patterns in pre-rule-engine event data

Continuously analyze the underlying event data to discover recurring relationships, lifecycle patterns and potentially useful behavior that fixed rules may not capture.

Examples include:

- Next-day amendments that cross an existing threshold, such as a booking around $9,000 being amended to around $10,000.
- Cancellation pairs with opposite directions and offsetting quantities.
- Related cancellation or late-event records that are not linked by the existing same-day logic.
- Desk-specific recurring patterns involving timing, instrument type, quantity, risk class or combinations of attributes.
- Patterns associated with alerts and patterns that occur without generating alerts.

These are examples and hypotheses, not guaranteed discoveries.

The system should provide business-relevant evidence and an understandable rationale for every proposed pattern. Merely saying that several records share attributes is insufficient.

The business should be able to validate, refine, reject, defer or revisit each candidate.

### Objective B: Improve supervisory alert review

Extend the existing SCP POC so that reviewers benefit from additional event-level context.

The solution should help supervisors:

- Understand the lifecycle leading to an alert.
- Find related events and potential episodes.
- Retrieve similar historical cases.
- Understand which attributes or events differ from prior cases.
- Receive evidence-based explanations and draft RFIs.
- Prioritize cases based on transparent evidence and expected review value.
- Reduce repetitive individual reviews through validated grouping.
- Use existing bulk-sign-off functionality for explicitly authorized batches where the business approves the approach.

Pattern discovery and alert investigation should be connected but remain distinct capabilities.

Discovering a pattern does not automatically make it an episode, prove that it is BAU, or authorize alert suppression.

---

## 5. Important distinction: fraud detection versus supervisory assistance

Do not frame this system as a conventional fraud-classification solution.

The primary business objective is to improve supervisory efficiency, discover and explain recurring trading behavior, provide alert context and support business decisions.

An anomaly score is not a fraud probability.

A recurring pattern is not automatically acceptable.

A frequent pattern is not automatically BAU.

An alert closure is not proof of normal behavior.

A lack of an RFI is not proof of normal behavior.

A cluster of similar events does not, by itself, establish a meaningful business relationship.

The platform may identify anomalous, unusual or unexplained behavior for investigation, but the separate fraud-reporting process remains outside scope.

---

## 6. Proposed high-level architecture

Build a hybrid architecture with the following connected layers.

### Layer 1: Data ingestion and immutable storage

Start with batch ingestion from Oracle DBaaS and historical Excel/CSV exports.

Validate schemas, identifiers, timestamps, duplicates, nulls and relevant business invariants.

Preserve the source records immutably, including provenance sufficient to trace a row to its source.

Use a Bronze layer for raw data, a Silver layer for validated canonical events and a feature layer for derived analytics.

Invalid records must be quarantined or otherwise handled explicitly. A duplicate must not be declared solely because two rows share several business attributes.

### Layer 2: Canonical event and lifecycle model

Define a canonical event contract based on verified fields.

Keep source IDs, event types, timestamps, trade identifiers and available lineage references.

Reconstruct lifecycle relationships separately from the raw event data.

Maintain the method, evidence and uncertainty associated with each link.

Where fields or parent references are absent, represent the limitation explicitly.

The canonical model should initially support the current source system. It should be designed so that source adapters can be added later without requiring all discovery logic to depend on source-specific column names.

### Layer 3: Feature engineering

Create features appropriate to the actual schema, such as:

- Time between events.
- Event order and lifecycle transitions.
- Quantity changes and quantity-offset relationships.
- Price changes and threshold crossings where price data is available.
- Cancellation/late/amendment frequencies.
- Recurrence and burstiness over defined windows.
- Differences between an event and its prior lifecycle state.
- Instrument-, desk-, book- and risk-class-specific context.
- Potential cross-trade relationships.
- Missing-lineage and data-quality indicators.

Do not create features that require unavailable source fields.

Do not derive relationships by unrestricted all-to-all event comparison. Use efficient candidate generation and appropriate blocking based on relevant attributes and time windows.

### Layer 4: Pattern discovery and detection

Use multiple complementary approaches rather than committing prematurely to one discovery algorithm.

**A. Deterministic and structured discovery**

Use known lifecycle relationships, verified ID lineage, explicit quantity-offset hypotheses and structured temporal or sequential pattern mining.

Purpose: find explainable and reproducible patterns, establish baseline performance and reduce avoidable complexity.

**B. Statistical detection**

Evaluate robust statistical methods and desk-/instrument-appropriate baselines where suitable.

Purpose: characterize deviations, distributional shifts and repeated behavior without assuming a fraud label.

**C. Unsupervised ML**

Evaluate methods such as ECOD, HBOS and Isolation Forest where they provide useful independent signals.

Potentially explore HDBSCAN, sequence embeddings, representation learning or graph-based approaches where the underlying data supports them.

Purpose: discover candidates that are not fully captured by known relationships or structured pattern templates.

These techniques are optional candidates. Do not deploy every algorithm merely to increase the apparent sophistication of the system.

**D. Cross-trade relationship analysis**

Investigate candidate relationships between different trade lineages, including offsetting quantity and opposite-direction patterns.

Compare the discovered relationships with the existing reconciliation rules and their limitations.

**E. Pattern comparison**

Compare current and historical windows, different appropriate desk baselines and pattern variants. Distinguish true structural relationships from accidental similarity or differences caused by data quality.

The system should measure which discovery approach contributes useful additional candidates, not assume that embeddings or clustering automatically discover business meaning.

### Layer 5: Evidence and candidate assessment

Preserve each detector's individual score and reason.

Build an evidence record for every candidate containing, where available:

- Machine-readable pattern definition.
- Supporting and contradicting examples.
- Number of independent occurrences.
- Frequency and distribution across time windows.
- Desk and instrument context.
- Relevant alert-generation outcomes.
- Comparison against a suitable baseline.
- Candidate event or episode relationships.
- Data-quality limitations.
- Model and feature versions.
- Uncertainty and potential alternative interpretations.

Separate three concepts:

1. Evidence quality.
2. Potential business relevance.
3. Review priority.

Do not conflate an anomaly score, model confidence or review priority with the probability that behavior is BAU.

### Layer 6: Pattern registry

Maintain versioned pattern definitions and their lifecycle.

A pattern definition is a reusable hypothesis.

A pattern instance is an occurrence of that definition in actual data.

An episode is a proposed grouping of specific events and potentially alerts.

An alert is an existing supervisory record generated through the existing rule engine.

One pattern may have many instances; one episode may contain multiple event and alert records. These entities should have separate IDs and clear relationships.

The registry should preserve definitions, discovery method, supporting evidence, first/last observed windows, statistics, drift history, feedback, reviewer decisions and applicability.

Do not delete prior versions when a pattern changes. A withdrawn or stale pattern should remain traceable.

### Layer 7: Episode correlation and case management

Use a dedicated correlation stage to associate detections and events with existing episodes or cases.

Consider verified lifecycle lineage, shared entities, complementary quantities, time proximity and repeated behavioral patterns.

Classify each proposed relationship as sufficiently supported, uncertain or unsupported.

If the association is uncertain, record a candidate link or keep cases separate instead of forcing a connection.

Persist stable case identifiers, individual detection evidence, relationship provenance, case revisions and prior dispositions.

Late or corrected events must trigger controlled reassessment when they materially change the evidence. Preserve the earlier decision and audit history.

### Layer 8: Existing AI investigation agent

Extend the existing SCP assistant rather than creating a completely separate agent without justification.

Retain its valuable functions:

- Retrieve similar closed cases.
- Compare prior cases with the current one.
- Summarize model and event evidence.
- Answer follow-up questions.
- Draft RFIs.
- Explain the attributes and events contributing to a case.

Add tools for:

- Retrieving pattern definitions and evidence.
- Inspecting authorized lifecycle records.
- Comparing historical time windows.
- Examining cross-trade relationships.
- Retrieving candidate and validated pattern instances.
- Calculating or querying deterministic pattern statistics.
- Retrieving applicable supervisor decisions and historical case context.
- Preparing proposed rule specifications for business review.

Agent operations must be bounded, auditable and based on authorized data. Use read-only evidence tools for investigation.

Do not allow the LLM to run unrestricted SQL, fabricate numerical findings, independently change Oracle rules, silently change the pattern registry or autonomously grant BAU status.

Keep model scoring and initial case creation independent of LLM availability.

### Layer 9: Grounded explanation and fallback

Produce an immediate deterministic explanation from the available structured evidence.

The agent can then create a business-readable narrative citing the evidence.

Validate the narrative against the evidence available to the system. Where appropriate, verify numeric claims using deterministic computation rather than trusting the LLM's arithmetic.

Allow a bounded retry for a failed narrative. If grounding fails, the model times out or the provider is unavailable, preserve the deterministic explanation.

Keep the case available to the supervisor even if AI investigation cannot complete.

### Layer 10: Business validation and supervisory workflow

Provide a business-facing pattern-review capability.

The sponsoring business team should review candidate patterns and route desk-specific patterns to appropriate supervisors.

Proposed decisions may include:

- Expected and approved within a stated scope.
- Needs refinement.
- Insufficient evidence.
- Rejected.
- Candidate for a future rule-engine change.

These categories are proposed and require business validation.

Maintain explicit reviewer decisions, rationale, timestamps, versions and applicability. Do not infer approval merely from an alert being closed.

A pattern approved by one desk should not automatically be treated as BAU for every other desk.

### Layer 11: Alert enrichment and operational improvement

Use validated event and pattern context to improve the existing alert review process.

Possible improvements include:

- Attach relevant event histories to alerts.
- Link supported relationships to episodes.
- Prioritize alerts with transparent, evidence-based scores.
- Surface similar historical cases and useful explanations.
- Reduce repetitive manual context gathering.
- Prepare groups for supervisor-authorized bulk review.
- Produce candidate rule specifications for the business.

The system must distinguish actual reduction in generated alerts from a reduction in review workload.

Generated alert volume changes only if an approved and correctly implemented change affects the upstream reconciliation, event transformation or rule-engine workflow. Pattern discovery alone does not reduce alert generation.

Actual changes to rules and thresholds remain business-controlled and manual initially.

### Layer 12: Feedback, model improvement and drift

Store structured business and supervisor decisions.

Use them to evaluate candidate usefulness and, when sufficient reliable examples exist, train a ranking or classification model to prioritize future candidates.

Do not assume that every feedback record is a clean label. Record the meaning, scope and quality of each label.

Monitor multiple forms of drift:

- Data-quality or schema drift.
- Event distribution drift.
- Lifecycle and relationship pattern changes.
- Changes in pattern frequency and stability.
- Changes in supervisor acceptance or rejection.
- Changes in usefulness or operational impact.

Distribution drift is not proof that behavior is suspicious or that a pattern has become invalid.

Use rolling-window discovery and approved historical comparison. Model or baseline updates must pass evaluation and governance rather than automatically changing the live detector.

---

## 7. BAU is a supervised business decision

BAU means business-as-usual behavior, but no universally reliable BAU definition currently exists for this use case.

Different desks may have different instruments, trading volumes, strategies, risk profiles and working practices.

The definition must therefore be desk- and context-aware, with explicit supervisory validation.

The ML system may discover recurring behavior and estimate whether a candidate is likely to be useful for review. It must not decide that recurring or statistically normal behavior is necessarily acceptable.

A supervisor-approved pattern should have a defined scope, applicable attributes and business context, effective period, supporting evidence and a record of its approval.

Do not use a single arbitrary clean-closure percentage as the definition of BAU. Historical outcomes may inform an assessment, but explicit business decisions and supporting evidence are essential.

Initially, support recommendations and supervisor-controlled batch actions rather than automatic BAU closure.

If automated closure is ever considered, require a validated target, shadow evaluation, explicitly authorized criteria, auditability, reversibility and continuing quality checks. Treat this as a separate future approval rather than an assumed feature.

---

## 8. Continuous processing and scalability

The existing process is batch-oriented. A live feed has not been confirmed.

Design the first version to work effectively in batch mode.

The proposed starting stack is:

- Python for analytics orchestration and model development.
- Polars for data cleaning, profiling and feature transformations.
- DuckDB for analytical SQL and joins.
- Parquet for the immutable historical archive and efficient analytical access.
- scikit-learn and/or PyOD for evaluated ML methods.
- The existing Java/SCP services for business workflow integration.
- An approved LLM endpoint and the current agent framework for investigations.

Consider MLflow for experiment tracking and model-version management, and Dagster for reproducible orchestration, retries and backfills where they add useful capabilities.

Only introduce a continuous-processing stack when operational requirements justify it:

- An approved continuous event feed or CDC connector.
- Kafka for durable event delivery and replay.
- Flink for event-time processing and stateful windows.
- PostgreSQL or an equivalent transactional store if existing storage does not adequately meet case-management needs.
- ClickHouse or another analytical serving store if actual query workloads require one.
- FastAPI and React/TypeScript if the desired integration or review UI requires them.

The existing application services and storage must be assessed before adding parallel systems.

Avoid unrestricted pairwise comparisons over millions of events. Use candidate blocking, temporal filters, incremental processing and well-defined analytical windows. Benchmark actual runtime, memory, database load and candidate volumes before choosing distributed infrastructure.

Streaming and low latency are future possibilities, not promises to make before the source feed and performance requirements are established.

---

## 9. Evaluation and validation methodology

The design must be validated empirically rather than defended because it contains advanced models.

Use historical time-based evaluation, separating earlier development windows from later test windows.

Compare at least:

1. A structured, deterministic discovery baseline.
2. An unsupervised ML discovery approach.
3. A combined approach.

Evaluate whether the ML methods uncover additional useful candidates rather than merely rediscovering the known reconciliation patterns.

Known examples include the possible offsetting cancellation pair and a next-day amendment crossing a threshold.

Where appropriate, use injected or carefully constructed test anomalies to verify that the technical pipeline behaves as expected. Clearly distinguish synthetic validation results from real business outcomes.

Measure:

- Candidate usefulness as rated by business reviewers.
- Proportion of reviewed candidates accepted or refined.
- New validated patterns discovered.
- Number of duplicate or unhelpful candidates.
- Coverage of relevant events and existing alerts.
- Case-linking quality and ambiguity.
- Time spent on supervisory review.
- RFI workload and repeated context-gathering effort.
- Actual alert-volume changes following approved rule changes.
- Batch runtime, memory use, database load and LLM cost.
- Stability across time windows and the frequency of misleading drift notifications.
- Data-quality failures and the correctness of late-event revisions.

Because all alerts eventually close in the supervisory workflow, closure rate alone is not an appropriate success metric.

Start with shadow mode: generate discoveries, evidence and recommendations without modifying production rules or autonomously closing cases. Measure their usefulness before introducing approved workflow changes.

---

## 10. Technical and operational safeguards

Treat the following as architectural invariants:

1. Source records are preserved and traceable.
2. Uncertain relationships are represented explicitly.
3. Raw data and derived interpretations are not conflated.
4. Every pattern, model, baseline, explanation and material case revision is versioned and auditable.
5. The LLM is not required for initial detection or score calculation.
6. No alert is silently discarded because an agent or queue fails.
7. Historical case decisions are not overwritten when late evidence arrives.
8. No business rule or Oracle threshold is changed autonomously.
9. No BAU designation is based on density, frequency, model score or closure status alone.
10. Business authorization remains separate from ML prediction.
11. Feedback is not automatically assumed to be a correct training label.
12. Confidential attachments remain outside the analysis unless access is explicitly authorized.
13. All AI tool calls, data access and any workflow actions follow approved access controls.
14. Model releases and baseline updates require evaluation and approval.
15. Performance and scalability claims must be supported by measurements.

---

## 11. Major architectural risks to resolve

Help me evaluate these points as we design the system.

**Risk A: Incomplete trade lifecycles.** Most records are cancellations and late events. We may not have every original booking record. Determine how to discover useful patterns without assuming complete histories.

**Risk B: Incorrect cross-trade relationships.** Opposing directions and offsetting quantities are candidate evidence, not proof that two records belong together. Define how relationships are supported, challenged and revised.

**Risk C: Unreliable pseudo-labels.** RFI-related labels and closure comments may reflect the existing review process. Establish how these can be used responsibly and when explicit supervisor labels become available.

**Risk D: Weak business explanations.** The LLM might invent a rationale for a statistical relationship. Require evidence-grounded explanations that distinguish observations from interpretation.

**Risk E: Unsupervised discovery that adds little value.** Embeddings and clustering may fail to discover useful relationships. Compare their outputs against simpler structured approaches.

**Risk F: Data and behavior drift.** Changes in distributions may indicate upstream data changes, different populations or genuine behavioral changes. Do not automatically classify every drift as suspicious.

**Risk G: Review overload.** A discovery system can overwhelm its own business reviewers with too many candidate patterns. Design transparent candidate prioritization and measure review capacity.

**Risk H: No direct alert reduction.** The event-analysis system operates before the current threshold-based rules, but the eventual operational path for implementing changes needs explicit approval. Distinguish recommendations from production rule changes.

**Risk I: Overengineering.** Do not add Kafka, Flink, graph databases, additional model registries or extra services until a measured requirement justifies them.

**Risk J: Unclear source-to-case integration.** Keep event IDs, alert IDs, pattern IDs, pattern-instance IDs, episode IDs and case IDs distinct. Define their relationships before implementing integration.

---

## 12. Decisions that need to be made collaboratively

Do not assume these are already settled.

- The exact source event contract and identifier conventions.
- Whether the existing dataset contains all relevant original booking events or only a subset.
- Which candidate patterns the business considers valuable.
- Which kinds of patterns should create or extend episodes.
- The initial discovery window and batch cadence.
- The preferred candidate-ranking method.
- Whether a new pattern-review interface is required or can be incorporated into the SCP portal.
- The storage location for the pattern registry and evidence.
- The required integration boundaries with the Java services.
- Which attributes can be compared across different desks.
- Which review outcomes should be captured as explicit training labels.
- What criteria would permit an approved pattern to participate in bulk sign-off.
- Whether and when a continuous data feed will exist.
- What runtime, latency, memory and database-load targets must be met.

For each decision, explain its significance, propose a sensible default, identify risks, and ask me to approve it before locking it into the design.

---

## 13. Recommended phased implementation

### Phase 1: Validate the data and event foundation

Define the provisional event contract, ingestion strategy, raw-data preservation, validation rules and lifecycle-link representation.

### Phase 2: Establish a discovery baseline

Implement structured lifecycle analysis, candidate cross-trade matching and basic statistical pattern discovery. Validate the known business examples and collect evidence.

### Phase 3: Evaluate additional ML methods

Compare statistical baselines, suitable unsupervised detectors and optional representation-learning approaches against the baseline. Keep only methods that provide measurable benefits.

### Phase 4: Build the pattern registry and review process

Store versioned candidates, supporting evidence, examples, counterexamples and business decisions. Establish a manageable review workflow for the sponsoring business and desk supervisors.

### Phase 5: Extend the existing SCP assistant

Provide controlled tools for event retrieval, pattern analysis, historical comparisons, evidence-grounded explanation and RFI drafting. Implement deterministic fallbacks and operational safeguards.

### Phase 6: Integrate supervisory intelligence

Enrich existing alerts, improve supported episode links, evaluate prioritization and pilot approved bulk-review workflows.

### Phase 7: Establish continuous improvement

Monitor drift, measure outcomes, collect explicit feedback and release updated models through governed evaluation.

### Phase 8: Scale the infrastructure when justified

Introduce continuous event processing and additional storage or serving infrastructure only after source availability and actual performance requirements are established.

These phases are a proposal, not an inflexible project plan. Refine them as dependencies and business requirements become clearer.

---

## 14. What I want from you now

First, summarize your understanding of this project in your own words and identify any contradictions or ambiguous requirements.

Then propose the logical architecture at a conceptual level, explaining how each component supports the business objectives.

Do not immediately generate the complete codebase or assume all technologies have been selected.

Start with the first unresolved architectural decision. Explain it thoroughly, discuss alternatives and trade-offs, recommend a provisional choice and wait for my approval.

Continue in this approval-driven manner throughout the design and eventual implementation.

The result should be a technically defensible, business-validated hybrid AI platform that reuses the existing SCP POC, discovers useful trading-event patterns, supports explainable supervisory decisions and demonstrably reduces review effort without silently compromising existing controls.



flowchart TD

```mermaid

    %% =========================================================
    %% 1. SOURCES AND CURRENT BUSINESS SYSTEMS
    %% =========================================================
    subgraph S["1. DATA SOURCES AND EXISTING BUSINESS SYSTEMS"]
        ORACLE["Oracle DBaaS<br/>Transformed trade events before rule evaluation"]
        FILES["Historical Excel / CSV exports"]
        RULES["Existing threshold-based rule engine<br/>Business-configured rules"]
        ALERTS["Existing rule-generated alerts"]
        SCP["Existing SCP<br/>Java workflow, data and blob services"]
        HISTORY["Existing alert and episode history<br/>RFI rounds, sign-off comments and permitted context"]
        LIVE["Future approved live feed or CDC<br/>Not confirmed; not required for MVP"]

        ORACLE --> RULES --> ALERTS --> SCP
        SCP --> HISTORY
    end

    %% =========================================================
    %% 2. INGESTION AND DATA FOUNDATION
    %% =========================================================
    subgraph D["2. INGESTION, DATA QUALITY AND CANONICAL EVENTS"]
        IMPORT["Batch importer<br/>Python and approved Oracle extraction"]
        VALIDATE{"Schema, timestamps, identifiers<br/>and record checks pass?"}
        QUARANTINE["Quarantine invalid records<br/>Record reasons and data-quality metrics"]
        BRONZE[("BRONZE<br/>Immutable raw archive<br/>Parquet / approved storage")]
        CLEAN["Cleaning and normalization<br/>Polars"]
        SILVER[("SILVER<br/>Validated canonical events")]
        SQL["Historical analytical queries and joins<br/>DuckDB"]
        CONTRACT["Canonical event contract<br/>Event type, source IDs, timestamps,<br/>trade identifiers and available attributes"]
        LINEAGE["Trade lifecycle reconstruction<br/>Verified lineage plus candidate links<br/>Preserve ambiguous or missing relationships"]
        FEATURES["Feature engineering<br/>Temporal, statistical, lifecycle,<br/>quantity and cross-trade features"]

        ORACLE --> IMPORT
        FILES --> IMPORT
        LIVE -.-> IMPORT
        IMPORT --> VALIDATE
        VALIDATE -->|Invalid| QUARANTINE
        VALIDATE -->|Valid| BRONZE
        BRONZE --> CLEAN
        CLEAN --> SILVER
        SILVER --> SQL
        SQL --> CONTRACT
        CONTRACT --> LINEAGE
        LINEAGE --> FEATURES
    end

    %% =========================================================
    %% 3. MULTI-METHOD DISCOVERY AND ML
    %% =========================================================
    subgraph M["3. EVENT PATTERN DISCOVERY AND ML DETECTION"]
        SEQUENCE["A. Lifecycle sequence mining<br/>Booking, amendment, cancellation,<br/>late events and timing transitions"]

        CROSS["B. Cross-trade relationship discovery<br/>Candidate matching, instrument and risk class,<br/>direction, quantity offset and time windows"]

        BEHAVIOR["C. Recurring desk-specific behavior<br/>Frequency, attribute combinations,<br/>temporal patterns and peer comparisons"]

        BASELINE["D. Robust statistical baselines<br/>Median, MAD, IQR, quantiles and<br/>appropriate desk or instrument context"]

        UNSUP["E. Optional unsupervised detectors<br/>ECOD, HBOS, Isolation Forest"]

        ADVANCED["F. Optional discovery experiments<br/>Sequence embeddings, HDBSCAN,<br/>graph or other learned representations"]

        FEATURES --> SEQUENCE
        FEATURES --> CROSS
        FEATURES --> BEHAVIOR
        FEATURES --> BASELINE
        FEATURES --> UNSUP
        FEATURES --> ADVANCED
    end

    %% =========================================================
    %% 4. EVIDENCE ASSEMBLY AND CASE CORRELATION
    %% =========================================================
    subgraph C["4. EVIDENCE, PATTERN CANDIDATES AND CASE FORMATION"]
        LEGACY["Existing alert POC signals<br/>XGBoost using RFI-derived proxy labels<br/>and Isolation Forest<br/>Retain as baselines; evaluate suitability"]

        EVIDENCE["Evidence assembly<br/>Preserve individual scores and reasons,<br/>supporting examples, counterexamples,<br/>frequency, context and uncertainty"]

        CANDIDATE["Candidate assessment<br/>Evidence quality, novelty, stability<br/>and potential business relevance"]

        PATREG[("VERSIONED PATTERN REGISTRY<br/>Definitions, evidence, examples,<br/>discovery method, statistics and feedback")]

        CORRELATE["Episode and case correlation<br/>Combine event lineage, detector evidence,<br/>existing links and relevant patterns"]

        LINK{"Is this relationship<br/>sufficiently supported?"}

        ATTACH["Attach to an existing episode<br/>or create a supported episode"]
        UNCERTAIN["Keep a candidate link<br/>and preserve uncertainty"]
        SEPARATE["Keep separate; create an independent<br/>case only when investigation is warranted"]

        CASESTORE[("CASE AND EPISODE STORE<br/>Stable IDs, relationships, revisions,<br/>evidence and disposition history")]

        OBJECTS["Keep these entities distinct<br/>Event ≠ Pattern ≠ Pattern instance<br/>≠ Episode / Case ≠ Alert"]

        SEQUENCE --> EVIDENCE
        CROSS --> EVIDENCE
        BEHAVIOR --> EVIDENCE
        BASELINE --> EVIDENCE
        UNSUP --> EVIDENCE
        ADVANCED --> EVIDENCE
        HISTORY --> LEGACY
        LEGACY --> EVIDENCE

        EVIDENCE --> CANDIDATE
        CANDIDATE --> PATREG

        LINEAGE --> CORRELATE
        EVIDENCE --> CORRELATE
        HISTORY --> CORRELATE
        CORRELATE --> LINK

        LINK -->|Supported| ATTACH
        LINK -->|Uncertain| UNCERTAIN
        LINK -->|Not established| SEPARATE

        ATTACH --> CASESTORE
        UNCERTAIN --> CASESTORE
        SEPARATE --> CASESTORE

        OBJECTS -.-> PATREG
        OBJECTS -.-> CASESTORE
    end

    %% =========================================================
    %% 5. EXISTING AI ASSISTANT EXTENDED
    %% =========================================================
    subgraph AIA["5. DETERMINISTIC EXPLANATION AND AGENTIC INVESTIGATION"]
        TEMPLATE["Immediate deterministic explanation<br/>Structured evidence and reason records"]

        QUEUE["Priority queue and bounded workers<br/>Prioritize criticality, uncertainty<br/>and expected investigative value"]

        CAPACITY{"Investigation capacity<br/>available?"}
        DEFER["Defer lower-priority work<br/>Keep case and deterministic explanation"]

        AGENT["Existing SCP investigation agent<br/>Extended with pattern-analysis tools"]

        TOOLS["Authorized read-only tools<br/>Event lifecycle, case evidence,<br/>pattern registry and historical context"]

        LLM["Approved LLM endpoint<br/>Investigate evidence, compare cases,<br/>draft narrative, assessment or RFI"]

        GROUND1{"Does the narrative<br/>match the evidence?"}
        RETRY["Regenerate once<br/>Maximum two narrative generations"]
        GROUND2{"Does the retry pass<br/>evidence validation?"}

        FALLBACK["Deterministic fallback<br/>Record failure or timeout"]
        EXPLANATION[("Versioned explanation record<br/>Evidence references, model and prompt versions")]
        
        CASESTORE --> TEMPLATE
        CASESTORE --> QUEUE
        PATREG --> QUEUE

        QUEUE --> CAPACITY
        CAPACITY -->|Available| AGENT
        CAPACITY -->|Overloaded| DEFER
        DEFER -.->|Resume when capacity returns| QUEUE

        AGENT <--> TOOLS
        CASESTORE -.-> TOOLS
        PATREG -.-> TOOLS
        HISTORY -.-> TOOLS
        SQL -.-> TOOLS

        AGENT --> LLM
        LLM --> GROUND1
        GROUND1 -->|Pass| EXPLANATION
        GROUND1 -->|Fail| RETRY
        RETRY --> GROUND2
        GROUND2 -->|Pass| EXPLANATION
        GROUND2 -->|Fail| FALLBACK
        AGENT -.->|Tool failure or timeout| FALLBACK

        TEMPLATE --> EXPLANATION
        DEFER --> EXPLANATION
        FALLBACK --> EXPLANATION
    end

    %% =========================================================
    %% 6. BUSINESS VALIDATION AND SUPERVISORY WORKFLOW
    %% =========================================================
    subgraph BIZ["6. BUSINESS VALIDATION AND OPERATIONAL ACTIONS"]
        REVIEW["Business pattern-review interface<br/>Evidence, examples, rationale,<br/>scope and historical comparison"]

        SPONSOR["Sponsoring business team<br/>Reviews candidate usefulness"]

        DESK["Relevant desk supervisor<br/>Validates desk-specific applicability"]

        PATDEC{"Pattern decision"}

        ACCEPT["Expected within defined scope<br/>Record explicit BAU approval"]
        REFINE["Refine definition or request<br/>more evidence"]
        REJECT["Reject or mark insufficient evidence"]
        RULESPEC["Prepare candidate rule specification<br/>Evidence, conditions and expected impact"]

        ENRICH["Alert enrichment and prioritization<br/>Attach event context, supported episodes<br/>and relevant validated pattern evidence"]

        PORTAL["SCP supervisor review<br/>Case evidence, explanation and history"]

        CASEDEC["Human supervisory decision<br/>Sign-off, request information or<br/>follow existing escalation procedure"]

        BULKBATCH["Prepare a proposed batch<br/>from explicitly approved scope"]

        AUTH{"Supervisor authorized<br/>this bulk action?"}
        BULK["Existing SCP bulk-sign-off workflow<br/>Record actions and audit history"]

        RULEAPPROVAL["Business reviews candidate rule change"]
        RULECHANGE["Authorized manual update<br/>to reconciliation logic or Oracle thresholds"]

        PATREG --> REVIEW
        REVIEW --> SPONSOR
        SPONSOR --> DESK
        DESK --> PATDEC

        PATDEC -->|Accept within scope| ACCEPT
        PATDEC -->|Refine| REFINE
        PATDEC -->|Reject or insufficient| REJECT
        PATDEC -->|Rule improvement candidate| RULESPEC

        REFINE --> REVIEW
        ACCEPT --> ENRICH
        REJECT --> ENRICH
        RULESPEC --> RULEAPPROVAL
        RULEAPPROVAL --> RULECHANGE

        CASESTORE --> ENRICH
        PATREG --> ENRICH
        ENRICH --> PORTAL
        EXPLANATION --> PORTAL
        PORTAL --> CASEDEC

        ACCEPT --> BULKBATCH
        ENRICH --> BULKBATCH
        BULKBATCH --> AUTH
        AUTH -->|Authorized| BULK
        AUTH -->|Not authorized| PORTAL
    end

    %% =========================================================
    %% 7. FEEDBACK, OFFLINE EVALUATION AND GOVERNANCE
    %% =========================================================
    subgraph GOV["7. CONTROLLED LEARNING, DRIFT AND GOVERNANCE"]
        FEEDBACK[("Auditable feedback store<br/>Pattern verdicts, scope, reasons,<br/>case decisions and reviewer identity")]

        CURATE["Feedback curation<br/>Separate explicit decisions from<br/>noisy RFI and sign-off signals"]

        PROFILE["Historical profiling and rolling-window analysis"]

        EXPERIMENT["Candidate discovery and model experiments<br/>Compare structured baselines, ML methods<br/>and combined approaches"]

        TEST["Chronological holdout evaluation<br/>Known examples, controlled synthetic tests,<br/>contamination and quality checks"]

        DRIFT["Monitor distinct changes<br/>Data quality, event distributions,<br/>pattern stability and validation outcomes"]

        GATE{"Candidate model or baseline<br/>approved for release?"}

        HOLD["Hold or reject candidate<br/>Investigate shortcomings"]

        MODELREG[("Versioned approved models<br/>and baseline snapshots")]

        RANKER["Future candidate-ranking model<br/>Only when enough reliable<br/>reviewer labels exist"]

        OPS["Operations and observability<br/>Data quality, runtime, database load,<br/>queue depth, drift, LLM failures and cost"]

        ALERTAUDIT["Audit and case revisions<br/>Preserve previous decisions and evidence"]

        CASEDEC --> FEEDBACK
        ACCEPT --> FEEDBACK
        REFINE --> FEEDBACK
        REJECT --> FEEDBACK
        BULK --> FEEDBACK
        RULECHANGE --> FEEDBACK

        FEEDBACK --> CURATE
        CURATE --> EXPERIMENT
        BRONZE --> PROFILE
        SILVER --> PROFILE
        PROFILE --> EXPERIMENT
        EXPERIMENT --> TEST
        TEST --> GATE

        GATE -->|Approved| MODELREG
        GATE -->|Rejected or uncertain| HOLD
        HOLD -.->|Revise candidate if justified| EXPERIMENT

        MODELREG -.->|Governed detector updates| BASELINE
        MODELREG -.->|Governed detector updates| UNSUP
        MODELREG -.->|Governed discovery updates| ADVANCED

        CURATE -.->|When labels are adequate| RANKER
        RANKER -.->|Candidate prioritization only| CANDIDATE

        MODELREG --> DRIFT
        PATREG --> DRIFT
        CASESTORE --> DRIFT
        IMPORT --> OPS
        EVIDENCE --> OPS
        AGENT --> OPS

        CASESTORE --> ALERTAUDIT
        ALERTAUDIT --> CASESTORE
    end

    %% =========================================================
    %% 8. LATE AND CORRECTED EVENTS
    %% =========================================================
    subgraph LATE["8. LATE EVENTS AND CASE CORRECTIONS"]
        ARRIVAL["Late, corrected or newly linked event"]
        RESOLVE["Resolve affected lifecycle and case<br/>using stable identifiers and evidence"]
        MATERIAL{"Does the new evidence<br/>materially change the case?"}
        REVISE["Append revision, recalculate evidence<br/>and re-evaluate under review policy"]
        NOCHANGE["Retain current decision<br/>Record the event and assessment"]

        SILVER --> ARRIVAL
        ARRIVAL --> RESOLVE
        RESOLVE --> MATERIAL
        MATERIAL -->|Yes| REVISE
        MATERIAL -->|No| NOCHANGE
        REVISE --> CASESTORE
        NOCHANGE --> CASESTORE
    end

    %% =========================================================
    %% 9. TECHNOLOGY MAPPING AND DEPLOYMENT BOUNDARIES
    %% =========================================================
    subgraph TECH["9. TECHNOLOGY CHOICES: MVP FIRST, SCALE WHEN JUSTIFIED"]
        MVP["CURRENT BATCH MVP<br/>Python, Polars, DuckDB, Parquet<br/>scikit-learn / PyOD<br/>Existing Java SCP and Oracle"]

        AGENTSTACK["AGENT AND APPLICATION<br/>Existing agent framework<br/>Approved LLM endpoint<br/>Existing SCP UI and APIs<br/>FastAPI or React only if needed"]

        GOVERNSTACK["OPTIONAL GOVERNANCE TOOLS<br/>MLflow for experiment/model tracking<br/>Dagster for orchestration and backfills"]

        FUTURE["CONDITIONAL FUTURE INFRASTRUCTURE<br/>Kafka + Flink only for an approved<br/>continuous event feed and measured need<br/>ClickHouse for justified analytics workloads<br/>PostgreSQL only if existing persistence is insufficient"]

        SECURITY["Cross-cutting controls<br/>Least privilege, approved data access,<br/>encryption, audit retention and<br/>versioned models, patterns and prompts"]
    end

    IMPORT -.-> MVP
    SQL -.-> MVP
    BASELINE -.-> MVP
    AGENT -.-> AGENTSTACK
    PORTAL -.-> AGENTSTACK
    MODELREG -.-> GOVERNSTACK
    PROFILE -.-> GOVERNSTACK
    LIVE -.-> FUTURE
    OPS -.-> SECURITY
    CASESTORE -.-> SECURITY
    EXPLANATION -.-> SECURITY
