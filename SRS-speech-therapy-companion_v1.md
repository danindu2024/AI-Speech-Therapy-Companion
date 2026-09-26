# Software Requirements Specification (SRS)
## AI-Powered Speech Therapy Companion

**Document version:** 1.0
**Date:** 2026-09-21
**Project reference:** Flagship project, Phase 2 — see `career-mentoring-guide_v2.md` (Section 6 & 8) and `flagship-project-tracker_v1.md`
**Prepared by:** [Your name]

> **Note on this draft:** This SRS is built from the project scoping already done in the master career guide (patient/therapist profiles, exercise library, Whisper-based feedback loop, progress dashboard, AWS serverless deployment). Sections marked *"[TBD — confirm]"* are reasonable first-pass assumptions rather than confirmed decisions — review and adjust before treating this as final, especially anything touching health-adjacent data handling.

---

## 1. Introduction

### 1.1 Purpose

This document specifies the functional and non-functional requirements for the **AI-Powered Speech Therapy Companion**, a web application that lets speech therapists assign structured articulation/pronunciation exercises to patients, lets patients record themselves completing those exercises, and uses an AI speech-to-text pipeline (Whisper) plus scoring logic to give feedback and track progress over time.

This SRS exists to:
- Define a single source of truth for what the system does before implementation deepens, so build effort (Phase 2 of the master plan) stays scoped.
- Serve as the reference document for architecture decisions, test planning, and the eventual "system design" narrative used in job interviews (per Section 5.3 of the master guide — production AI handling is a stated differentiator, and a clear SRS is part of demonstrating engineering rigor to local employers, per Section 5.2).
- Give any future contributor (including an AI assistant helping with later phases) full context without needing the original conversation.

### 1.2 Scope

The system covers:
- Patient and therapist account management.
- A therapist-curated exercise library.
- Assignment of exercises from therapists to patients.
- Patient-side audio recording of exercise attempts.
- AI-assisted transcription (Whisper) and articulation/pronunciation scoring of recordings.
- Progress tracking and a dashboard view for both patient and therapist.
- Deployment as a cloud-hosted (AWS serverless) web application with production-grade handling of AI latency and cost (Section 5.3 of the master guide).

The system explicitly does **not** cover (out of scope for the MVP — see Section 13, Alternative Solutions Considered, and Section 7, Constraints, for why):
- Training a custom speech-recognition or scoring model from scratch — the project is an *application layer* over an existing model (Whisper), not an ML research project (per Section 8 of the master guide's scoping principle).
- Real-time, in-session video/audio therapy calls (this is an asynchronous exercise-and-feedback tool, not a telehealth video platform).
- Native mobile apps (web-first; responsive web is in scope, native iOS/Android is not, consistent with Section 3 of the master guide deliberately not pursuing mobile in parallel).
- Billing/payment processing (the MVP assumes therapists and patients are already provisioned, e.g. by a clinic admin — no self-serve paid signup flow).
- Multi-language exercise content beyond one initial language *[TBD — confirm; assumed English-only for MVP, see Section 8]*.

### 1.3 Definitions, Acronyms, and Abbreviations

| Term | Definition |
|---|---|
| **SRS** | Software Requirements Specification (this document) |
| **MVP** | Minimum Viable Product |
| **Patient** | End user completing speech therapy exercises |
| **Therapist** | End user who creates/assigns exercises and reviews patient progress |
| **Exercise** | A structured speech/articulation task assigned to a patient |
| **Session** | One instance of a patient recording and submitting an exercise attempt |
| **Whisper** | OpenAI's speech-to-text model, used here for transcription of patient recordings |
| **Scoring engine** | The custom logic that evaluates a transcription against the target exercise for pronunciation/articulation accuracy |
| **Lambda** | AWS Lambda — serverless compute used for backend processing |
| **DynamoDB** | AWS's managed NoSQL database, used as the primary data store |
| **Cognito** | AWS Cognito — used for authentication/user pool management |
| **SQS** | AWS Simple Queue Service — used for async processing of transcription jobs |
| **S3** | AWS Simple Storage Service — used for audio file and static asset storage |
| **CloudFront** | AWS's CDN, used to serve the frontend and/or cached audio |
| **PII** | Personally Identifiable Information |
| **PHI-adjacent data** | Health-related personal data (patient speech recordings, therapy progress); treated with elevated care even though this is not a formal clinical/regulated deployment |
| **RTO / RPO** | Recovery Time Objective / Recovery Point Objective (used in Section 5, Non-Functional Requirements) |

---

## 2. Project Scope

*(Product perspective — how this system fits into its environment.)*

The Speech Therapy Companion is a **standalone web application**, not an extension of an existing system. It is built as a portfolio-grade flagship project (per the master career guide) but is designed to be realistic enough to plausibly serve a small speech-therapy practice: one or more therapists, each managing multiple patients.

**Product perspective:**
- Greenfield build — no legacy system integration required.
- Single-tenant assumption for MVP: one deployed instance serves one practice's therapists and patients. Multi-tenancy (multiple unrelated clinics on one deployment) is a stated future enhancement, not an MVP requirement.

**Product functions (summary — detailed in Section 4):**
1. Account management & authentication (patient, therapist roles).
2. Exercise library management (therapist creates/edits/organizes exercises).
3. Exercise assignment (therapist → patient).
4. Recording & submission (patient records an attempt at an assigned exercise).
5. AI transcription & scoring (system transcribes and scores the recording).
6. Feedback delivery (patient sees score/feedback; therapist sees the same plus aggregate trends).
7. Progress dashboard (visual progress over time, per patient, and per-therapist across their caseload).

**User classes:** see Section 3 (Stakeholders).

**Operating environment:** Modern web browsers (desktop and mobile web) with microphone access; AWS-hosted backend; no on-premise component.

---

## 3. Stakeholders

| Stakeholder | Role / Interest |
|---|---|
| **Patient** | Primary end user completing exercises; wants clear, low-friction recording and understandable feedback. |
| **Therapist** | Primary end user assigning exercises and monitoring progress; wants an efficient way to manage a caseload and see meaningful trend data, not just raw scores. |
| **Clinic admin** *[TBD — confirm if in MVP scope]* | Would provision therapist/patient accounts if a formal onboarding flow is needed; for MVP this role may be handled manually/out-of-band rather than as a built feature. |
| **Project owner / developer** | Builds and maintains the system; also uses it as a portfolio artifact for job applications (Section 5.2 of the master guide — architecture and live URL matter to this stakeholder specifically). |
| **Prospective employer (indirect stakeholder)** | Not a system user, but a consumer of the *artifacts* the system produces — architecture diagrams, case study write-ups, live demo. Included here because it legitimately shapes some requirements (e.g., the system should be demo-able with seeded sample data, Section 6). |

---

## 4. Functional Requirements

Requirements are grouped by module and numbered `FR-<module>-<n>` for traceability.

### 4.1 Account Management & Authentication
- **FR-AUTH-1:** The system shall allow a user to register as either a Patient or a Therapist. *[TBD — confirm: open self-registration vs. therapist-invites-patient only; the latter is more realistic clinically]*
- **FR-AUTH-2:** The system shall authenticate users via AWS Cognito (email/password at minimum).
- **FR-AUTH-3:** The system shall enforce role-based access: a Patient shall not access another patient's data; a Therapist shall only access data for patients under their own caseload.
- **FR-AUTH-4:** The system shall support password reset via email.
- **FR-AUTH-5:** The system shall allow a Therapist to invite a Patient by email, creating a pending account the patient activates.

### 4.2 Exercise Library Management
- **FR-EXL-1:** The system shall allow a Therapist to create an exercise with: title, description, target phoneme/word/phrase set, difficulty level, and optional reference audio.
- **FR-EXL-2:** The system shall allow a Therapist to edit or archive (not hard-delete) an exercise.
- **FR-EXL-3:** The system shall allow a Therapist to categorize/tag exercises (e.g., by sound, difficulty, or therapy goal) for filtering.
- **FR-EXL-4:** The system shall allow a Therapist to browse the exercise library and filter by tag/difficulty.

### 4.3 Exercise Assignment
- **FR-ASN-1:** The system shall allow a Therapist to assign one or more exercises to a specific Patient, optionally with a due date and target repetition count.
- **FR-ASN-2:** The system shall notify the Patient (in-app, minimum; email optional) when a new exercise is assigned. *[TBD — confirm: email notifications may be a post-MVP enhancement]*
- **FR-ASN-3:** The system shall allow a Therapist to view all currently assigned (not yet completed) exercises per patient.

### 4.4 Recording & Submission
- **FR-REC-1:** The system shall allow a Patient to record audio directly in-browser for an assigned exercise.
- **FR-REC-2:** The system shall allow a Patient to review and re-record before submitting.
- **FR-REC-3:** The system shall upload the recording to S3 and create a corresponding session record.
- **FR-REC-4:** The system shall support recordings up to [TBD — e.g., 2 minutes] in length for the MVP.

### 4.5 AI Transcription & Scoring
- **FR-AI-1:** The system shall transcribe each submitted recording using Whisper.
- **FR-AI-2:** The system shall process transcription asynchronously (not blocking the patient's UI) using either WebSocket streaming or an SQS-based queue (architecture decision tracked in `flagship-project-tracker_v1.md`, Section 4/5 — satisfies Section 5.3 of the master guide).
- **FR-AI-3:** The system shall run the transcription result through a custom scoring engine that compares it against the exercise's target phoneme/word/phrase set and produces a pronunciation/articulation score.
- **FR-AI-4:** The system shall apply prompt optimization and/or caching to avoid redundant AI processing costs on repeated or near-identical evaluations (Section 5.3 of the master guide).
- **FR-AI-5:** The system shall notify the Patient when scoring is complete (given the async nature of FR-AI-2).
- **FR-AI-6:** The system shall gracefully handle transcription failures (e.g., inaudible recording, service timeout) with a clear, actionable message rather than a silent failure — see also NFR-REL-2.

### 4.6 Feedback & Progress Tracking
- **FR-PRG-1:** The system shall display the score and any generated feedback text to the Patient after processing.
- **FR-PRG-2:** The system shall log every completed session (exercise, date, score) to a per-patient progress history.
- **FR-PRG-3:** The system shall provide a Patient-facing dashboard showing progress over time (e.g., score trend per exercise/category).
- **FR-PRG-4:** The system shall provide a Therapist-facing dashboard summarizing progress across their full caseload, with drill-down to an individual patient's history.
- **FR-PRG-5:** The system shall allow a Therapist to leave manual notes on a patient's session, supplementing the automated score.

---

## 5. Non-Functional Requirements

| ID | Category | Requirement |
|---|---|---|
| NFR-PERF-1 | Performance | The initial API response after a recording submission (i.e., acknowledgment of receipt) shall return in under 1 second, regardless of Whisper processing time — enforced by the async design in FR-AI-2. |
| NFR-PERF-2 | Performance | End-to-end feedback latency (submission → score available) shall be measured and tracked (see `flagship-project-tracker_v1.md`, Section 5) with a target of under [TBD — e.g., 15 seconds] for a typical short recording. |
| NFR-SCAL-1 | Scalability | The system shall use serverless compute (Lambda) so it scales to zero at idle and handles bursty load (e.g., many patients submitting near a due date) without manual capacity planning. |
| NFR-SEC-1 | Security | All data in transit shall use HTTPS/WSS. |
| NFR-SEC-2 | Security | All audio recordings and personal data at rest (S3, DynamoDB) shall be encrypted. |
| NFR-SEC-3 | Security | Access control shall be enforced at the API layer (not just hidden in the UI) — a Patient's API calls shall be rejected server-side if they attempt to access another patient's data. |
| NFR-SEC-4 | Security | Speech recordings are treated as sensitive (PHI-adjacent) data even though this is not a regulated clinical deployment; access shall be logged, and recordings shall not be publicly readable from S3. |
| NFR-AVAIL-1 | Availability | The system should target reasonable availability appropriate to a portfolio/demo deployment; formal SLA/uptime guarantees are out of scope for MVP *[TBD — confirm if a specific % is wanted for interview-narrative purposes]*. |
| NFR-USE-1 | Usability | The recording flow shall be completable by a non-technical user (patient) without instructions beyond in-app guidance — minimize steps between "assigned exercise" and "recording." |
| NFR-USE-2 | Usability | The application shall be responsive and usable on both desktop and mobile web, since patients may complete exercises on a phone. |
| NFR-MAINT-1 | Maintainability | Backend Lambda functions shall share common utility/auth layers rather than duplicating logic per function (per the build tracker's Week 4 note). |
| NFR-COST-1 | Cost efficiency | AI processing costs shall be actively managed via caching/prompt optimization (FR-AI-4) rather than left unconstrained — direct requirement from Section 5.3 of the master guide. |
| NFR-OBS-1 | Observability | Key pipeline stages (submission, transcription start/end, scoring) shall emit CloudWatch metrics/logs sufficient to diagnose a failed or slow session after the fact. |
| NFR-REL-1 | Reliability | Transcription job failures shall be retried a bounded number of times before surfacing an error to the user (avoids both silent data loss and infinite retry loops). |
| NFR-REL-2 | Reliability | The system shall not lose a submitted recording even if scoring fails — the raw recording and a "processing failed" state shall persist so a retry is possible without the patient re-recording. |

---

## 6. Business Rules

- **BR-1:** A Patient can only be created/invited by a Therapist (no fully open self-registration for patients) — reflects the real-world clinical relationship. *[TBD — confirm against FR-AUTH-1/5]*
- **BR-2:** An Exercise must belong to exactly one Therapist (its creator/owner) but may be assigned to any of that therapist's patients.
- **BR-3:** A Patient may only view and submit recordings for exercises explicitly assigned to them.
- **BR-4:** A completed session's score, once generated, is immutable — a re-attempt creates a new session record rather than overwriting the previous one, preserving the progress history (supports FR-PRG-2/3).
- **BR-5:** Only the assigning Therapist (or another therapist explicitly given access to that patient — post-MVP) may view a given patient's detailed session history.
- **BR-6:** An archived Exercise (FR-EXL-2) remains visible in historical session records (for progress history integrity) even though it can no longer be newly assigned.

---

## 7. Constraints

- **C-1 (Timeline):** The build window is constrained to Phase 2 of the master roadmap (Oct–Nov 2026), overlapping with AWS SAA exam prep (hard deadline Nov 14, 2026) and university coursework — see `flagship-project-tracker_v1.md`.
- **C-2 (Team size):** Single developer. No dedicated QA, design, or DevOps role — all requirements must be achievable by one person within available weekly hours (~15–18 hrs/week project + ~10–12 hrs/week AWS-hands-on, per the master guide's Section 11).
- **C-3 (Technology stack):** Must use React + Vite (frontend), Node.js + TypeScript on AWS Lambda (backend), and DynamoDB (data) per Section 6 of the master guide — this is a deliberate constraint to bridge existing skills into the target market's stack, not an open technology choice.
- **C-4 (Budget):** AWS costs should stay within free-tier or minimal-cost bounds where feasible; Whisper/LLM API usage costs must be actively controlled (ties to NFR-COST-1).
- **C-5 (Scope discipline):** The system must remain an *application layer* over existing AI models — training or fine-tuning a custom speech model is explicitly out of scope (per Section 8 of the master guide), regardless of how it might affect scoring accuracy.
- **C-6 (Regulatory):** This is a portfolio project, not a certified medical device or regulated clinical tool — it must not represent itself as providing a clinical diagnosis, even though it handles health-adjacent data with care (NFR-SEC-4).

---

## 8. Assumptions

- **A-1:** Users have a device with a working microphone and a modern browser supporting the Web Audio/MediaRecorder APIs.
- **A-2:** Users have a stable-enough internet connection to upload short audio clips; offline recording/sync is not assumed.
- **A-3:** The MVP targets English-language exercises only; multi-language support is a future enhancement.
- **A-4:** Therapists are reasonably tech-comfortable (they are expected to build/manage an exercise library through a web UI without hand-holding).
- **A-5:** Whisper's transcription accuracy is sufficient for the MVP's scoring approach; scoring logic accounts for reasonable transcription noise rather than assuming perfect transcripts.
- **A-6:** Patient/therapist accounts are provisioned manually or via therapist invite for the MVP — there is no assumption of a large self-serve public user base.
- **A-7:** The deployed system is a demo/portfolio-grade instance, not expected to handle production clinical patient volumes.

---

## 9. Main Use Case Flows

### UC-1: Patient Completes an Assigned Exercise
**Actor:** Patient
**Preconditions:** Patient is authenticated; at least one exercise is assigned to them.
**Main flow:**
1. Patient opens their dashboard and sees a list of assigned exercises.
2. Patient selects an exercise and views its instructions/reference audio (if any).
3. Patient records their attempt in-browser.
4. Patient reviews the recording and confirms submission.
5. System uploads the recording (S3) and creates a session record in a "processing" state.
6. System queues the recording for transcription (async — FR-AI-2).
7. System transcribes (Whisper) and scores (custom logic) the attempt.
8. System updates the session record with the score/feedback and notifies the Patient.
9. Patient views their score and feedback; progress dashboard updates.

**Alternate flows:**
- 3a. Patient re-records before submitting (FR-REC-2) — loop back to step 3.
- 7a. Transcription fails → session marked "processing failed," retried per NFR-REL-1; if still failing, Patient is notified with an option to re-submit (NFR-REL-2).

**Postconditions:** A new, immutable session record exists (BR-4); patient and therapist progress views reflect it.

### UC-2: Therapist Assigns an Exercise
**Actor:** Therapist
**Preconditions:** Therapist is authenticated; has at least one patient and one exercise available.
**Main flow:**
1. Therapist opens the exercise library and selects an exercise (or creates a new one, FR-EXL-1).
2. Therapist selects one or more patients to assign it to, with optional due date/repetition target.
3. System creates assignment record(s) and notifies patient(s) (FR-ASN-2).
4. Therapist can view the assignment's status (not started / in progress / completed) from their dashboard.

**Postconditions:** Assigned exercise(s) appear on the relevant patient's dashboard (UC-1, step 1).

### UC-3: Therapist Reviews Patient Progress
**Actor:** Therapist
**Preconditions:** Therapist has at least one patient with completed sessions.
**Main flow:**
1. Therapist opens their caseload dashboard (FR-PRG-4).
2. Therapist selects a patient to view detailed history.
3. System displays score trends over time, per exercise/category.
4. Therapist optionally adds a manual note to a specific session (FR-PRG-5).

**Postconditions:** None state-changing beyond an optional note; primarily a read flow.

*(Additional use cases — e.g., "Therapist Creates/Edits Exercise," "Patient Handles a Failed Recording" — should be added here as they're fleshed out during Phase 2 build; this section is intentionally not exhaustive for v1 of the SRS.)*

---

## 10. User Activity Flows

**Patient activity flow (happy path):**
```
Login → View assigned exercises → Select exercise → View instructions
   → Record attempt → Review recording → Submit
   → [Async: wait for processing] → View score/feedback → View progress dashboard
```

**Therapist activity flow (happy path):**
```
Login → View caseload dashboard → [Create/edit exercise] OR [Assign existing exercise to patient(s)]
   → (time passes as patients complete exercises)
   → Review patient progress → Add notes as needed
```

**Failure-handling sub-flow (recording/processing):**
```
Submit recording → Upload to S3 → Queue for transcription
   → Transcription fails → Bounded retry (NFR-REL-1)
   → Still failing → Notify patient, preserve recording (NFR-REL-2) → Patient may retry submission
```

*(These are intentionally kept as text/flow diagrams for v1; swim-lane or activity diagrams can be added as an appendix once the UI is further along — see `flagship-project-tracker_v1.md`, Week 9, "architecture diagram finalized.")*

---

## 11. External Interface Requirements

### 11.1 User Interfaces
- Responsive web application (desktop + mobile web), React + Vite frontend.
- Distinct dashboard views for Patient and Therapist roles (FR-PRG-3/4).
- In-browser audio recording UI (FR-REC-1/2).

### 11.2 Hardware Interfaces
- Client-side microphone access (via browser MediaRecorder API) — no custom hardware integration.

### 11.3 Software Interfaces
- **AWS Cognito** — authentication (FR-AUTH-2).
- **AWS S3** — audio file storage, and static asset hosting for the frontend.
- **AWS CloudFront** (or Amplify) — CDN for frontend delivery.
- **AWS Lambda** — backend compute for all API logic.
- **AWS DynamoDB** — primary data store (profiles, exercises, assignments, sessions).
- **AWS SQS** and/or **WebSocket API Gateway** — async processing channel for transcription jobs (FR-AI-2; specific choice tracked in `flagship-project-tracker_v1.md`).
- **Whisper (speech-to-text model/API)** — transcription (FR-AI-1).
- **AWS CloudWatch** — logging/metrics (NFR-OBS-1).

### 11.4 Communication Interfaces
- HTTPS for all REST API calls.
- WSS (WebSocket Secure) if the WebSocket-streaming latency-mitigation approach is chosen over SQS (decision pending — see `flagship-project-tracker_v1.md`, Week 6).

---

## 12. Data Requirements

### 12.1 Core Entities (high-level — full schema lives in the build tracker's AWS Infrastructure Build Log)

| Entity | Key Attributes | Notes |
|---|---|---|
| **User** | userId, role (Patient/Therapist), name, email, createdAt | Cognito handles credentials; DynamoDB holds profile data keyed to Cognito `sub`. |
| **Exercise** | exerciseId, therapistId (owner), title, description, targetContent, difficulty, tags, status (active/archived) | BR-2, BR-6. |
| **Assignment** | assignmentId, exerciseId, patientId, therapistId, dueDate, repetitionTarget, status | FR-ASN-1/3. |
| **Session** | sessionId, assignmentId, patientId, recordingS3Key, transcript, score, feedbackText, status (processing/complete/failed), createdAt | BR-4 — immutable once complete; FR-PRG-2. |
| **Note** | noteId, sessionId, therapistId, text, createdAt | FR-PRG-5. |

### 12.2 Data Retention & Privacy
- Audio recordings (S3) and derived transcripts are treated as sensitive data (NFR-SEC-4); access is role-restricted (BR-3/5) and logged.
- Retention policy *[TBD — confirm]*: e.g., recordings retained indefinitely for progress history vs. a defined retention window — needs an explicit decision given the health-adjacent nature of the data (C-6).

### 12.3 Data Volume Expectations
- MVP/demo scale: low volume (single-digit therapists, tens of patients, hundreds of sessions) — sized for a portfolio deployment, not production clinical load (A-7), which keeps DynamoDB/Lambda well within free-tier-friendly usage.

---

## 13. Alternative Solutions Considered

| Alternative | Considered For | Why Rejected (for this project's goals) |
|---|---|---|
| Training/fine-tuning a custom speech-recognition model | Transcription | Out of scope per the project's own scoping principle (Section 8 of master guide) — the in-demand, demonstrable skill here is *integrating and reasoning about* AI systems, not ML research; also infeasible on the available timeline/budget. |
| Monolithic backend (e.g., a single Express server on EC2) | Overall architecture | Rejected in favor of serverless (Lambda) specifically because the project's purpose includes demonstrating cloud-native/AWS competency (Section 6 of master guide) and reinforcing AWS SAA study (Section 5.4). |
| Relational database (MySQL, already known) | Primary data store | Rejected in favor of DynamoDB deliberately — using only what's already known would forgo a genuinely sought-after NoSQL competency and the chance to demonstrate access-pattern-driven schema design (Section 6 of master guide). |
| Synchronous transcription (block the HTTP request until Whisper returns) | AI processing | Rejected — by early 2027 a plain blocking API wrapper is "standard backend engineering," not a differentiator (Section 5.3); async handling was made a first-class requirement (FR-AI-2). |
| Native mobile app (iOS/Android) | Client platform | Rejected for MVP — mobile development is explicitly deferred to avoid split focus (Section 3 of master guide); responsive web is used instead. |
| Real-time video telehealth session | Overall product concept | Rejected — significantly larger scope (WebRTC, scheduling, live session infra) than an asynchronous exercise-and-feedback tool; doesn't fit the available build window (C-1). |

---

## 14. Feasibility Study

### 14.1 Technical Feasibility
- Frontend (React) and general full-stack patterns are already within the developer's existing skill set (career guide Section 1), minimizing new-framework learning time.
- Backend/cloud pieces (TypeScript on Lambda, DynamoDB, Cognito) are new but are being learned in direct service of AWS SAA prep (Section 5.4) — the "hands-on = dual purpose" approach makes this feasible within available hours rather than requiring separate learning time.
- Whisper integration is a well-documented, widely-used API — technically low-risk compared to, say, building a custom model.
- **Assessment: Feasible**, contingent on the async-processing and cost-control pieces (Section 5.3) being scoped realistically rather than gold-plated.

### 14.2 Operational Feasibility
- Single developer will also act as sole "operator" post-deployment — no dedicated ops team. Serverless architecture (auto-scaling, no server patching) is specifically chosen to minimize ongoing operational burden (C-2, NFR-SCAL-1).
- **Assessment: Feasible** for a demo/portfolio-scale deployment (A-7); would need re-assessment before any real clinical use.

### 14.3 Economic Feasibility
- Primary costs: AWS usage (mostly free-tier-eligible at this scale) and Whisper/LLM API calls (the main variable cost, directly addressed by NFR-COST-1/FR-AI-4).
- **Assessment: Feasible** within a portfolio-project budget, provided cost controls (caching, prompt optimization) are implemented early rather than retrofitted — this is why Section 5, Week 7 of the build tracker treats cost management as its own milestone rather than an afterthought.

### 14.4 Schedule Feasibility
- The project must be built and deployed within Phase 2 (Oct–Nov 2026), overlapping with AWS SAA exam prep and preceding Phase 3's active application launch (C-1).
- Risk: this is a tight, single-developer timeline for a project with a genuinely non-trivial async/AI component. Mitigated by the phased weekly build plan in `flagship-project-tracker_v1.md`, which sequences plumbing (auth/data) before the harder AI-pipeline work, and by keeping the MVP scope disciplined (Section 7, C-5).
- **Assessment: Feasible but tight** — schedule risk is real and is carried forward into Section 15 below.

---

## 15. Risk Analysis

| ID | Risk | Probability | Impact | Mitigation |
|---|---|---|---|---|
| R-1 | AWS/serverless learning curve slows feature delivery, threatening the Phase 2 timeline | Medium | High | Hands-on AWS learning is scheduled as project-build time itself (Section 5.4 of master guide), not a separate track; weekly checklist in the build tracker surfaces slippage early. |
| R-2 | Whisper/LLM API costs exceed budget if usage isn't controlled | Medium | Medium | Caching and prompt optimization built as an explicit Week 7 milestone (FR-AI-4, NFR-COST-1), not deferred to "later if needed." |
| R-3 | Async processing (SQS/WebSocket) introduces bugs/complexity beyond a solo developer's available debugging time | Medium | Medium | Decision made explicitly and early (Week 6 of build tracker) rather than bolted on late; before/after latency is measured so problems are caught with data, not guesswork. |
| R-4 | Single point of failure: sole developer — illness, university exam conflicts, or burnout stalls the project | Medium | High | Timeline explicitly reserves Dec 2026–Feb 2027 as "maintenance mode only" around exams (Section 7 of master guide); scope discipline (C-5) keeps the MVP small enough to be resumable after a gap. |
| R-5 | Scope creep — adding features (multi-language, native mobile, billing) beyond MVP eats the build window | Medium | Medium | Explicit out-of-scope list (Section 1.2) and constraint C-5; any addition should be checked against this SRS before being started. |
| R-6 | Handling health-adjacent data (patient speech recordings) without adequate access control/encryption creates a privacy/reputation risk, even in a non-clinical demo | Low–Medium | High | NFR-SEC-2/3/4 and BR-3/5 specify encryption and strict role-based access as non-negotiable, not "nice to have," from the start. |
| R-7 | Whisper transcription accuracy is inconsistent for certain speech patterns (which is somewhat the point for a speech-therapy tool — disordered speech may transcribe poorly), undermining scoring reliability | Medium | Medium | Scoring engine designed to account for transcription noise (Assumption A-5) rather than trusting raw transcripts; flagged as an area to validate early with real sample recordings, not assumed correct by default. |
| R-8 | Demo-readiness risk: project works locally/in pieces but isn't presentable (stable live URL, clean architecture diagram) in time for Phase 3 applications | Medium | High | Portfolio packaging (Section 6 of the build tracker) is tracked from the start, not left until the week applications go out; Week 9 of the build plan is dedicated to hardening + demo-readiness, not new features. |

---

## Document Control

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-09-21 | Initial complete draft, derived from `career-mentoring-guide_v2.md` (Sections 5–8) and structured per the chapter list used in the author's prior university project SRS. |

*Companion documents: `career-mentoring-guide_v2.md` (overall career plan), `DSA_v2.md` (Phase 1 execution), `flagship-project-tracker_v1.md` (Phase 2 execution).*
