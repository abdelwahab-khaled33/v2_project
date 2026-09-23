# Product Requirements Document (PRD)
## University Exam & Quiz Platform

| | |
|---|---|
| **Status** | Draft |
| **Owner** | Abdelwahab |
| **Type** | Graduation project |
| **UI Language** | English |
| **Brand colors** | Primary Blue `#455B8A` · Accent Orange `#F2842F` · White `#FFFFFF` |

---

## 1. Problem Statement

The university currently has no unified digital system for creating, administering, and grading exams and quizzes across departments. Exam creation, question management, student data, and grading are handled manually or through disconnected tools. This creates overhead for Doctors and Teaching Assistants (TAs), inconsistent security during exams, and no central place for Admins to oversee and audit academic assessments.

## 2. Goal

Build a single web platform where an Admin can onboard all university users and subjects, Doctors and TAs can build question banks and run exams/quizzes under one governed workflow, and Students can take secure, timed, locked-down exams — reliable enough to be used for real, audited assessments (Midterms/Finals), not just practice tools.

## 3. Objectives / Success Criteria

- All four roles (Admin, Doctor, TA, Student) can complete their full workflow without leaving the platform.
- Doctor-authored high-stakes exams (Midterm/Final) go through a mandatory Admin approval gate before students can see them.
- Every student gets a unique, randomly-generated question set matching a fixed difficulty mix, reducing the value of copying a neighbor's screen.
- Exam sessions are locked down (Safe Exam Browser + fullscreen + disabled copy/paste) to reduce cheating opportunities.
- The system can withstand a stress scenario of ~5,000 students taking exams at the same instant, even though this isn't the expected everyday load.
- Grades are trustworthy enough to stand as the official record — no student can submit with unanswered questions, no student can see answers after submitting, and every grade override is explicit and attributable.

## 4. Scope

### In scope (v1)
- Admin: bulk user/subject onboarding via Excel, permission management, full exam oversight and approval.
- Doctor: private question bank per subject, exam creation/editing (pre-approval), results viewing/export, grade compensation for flawed questions.
- TA: question bank shared among all TAs of a subject, quiz creation for their own sections only (no approval needed), grade viewing/export scoped to their own students.
- Student: subject-scoped exam visibility, locked-down exam-taking experience, flagging, mandatory full completion before submit.
- Security: Safe Exam Browser integration via Browser Exam Key validation; fullscreen lock; copy/paste disabled.
- Question types: Multiple Choice and True/False only.
- Difficulty tiering: 3 fixed levels (Easy / Medium / Hard), fixed mix per exam.
- Excel-based bulk import for both users and questions, with a validation/dry-run step before committing.
- Results export to Excel (per exam, per quiz, or combined).

### Out of scope (v1)
- SSO / direct integration with the university's central student information system — accounts are Excel-only for now.
- Question types beyond MCQ and True/False (e.g. essay, fill-in-the-blank).
- Per-student adaptive difficulty (the mix is fixed per exam, not adjusted per student ability).
- A dedicated "compensatory exam" system feature — a redo is just a normal new exam; the university manages student device access for it outside the system.
- Cross-device session handoff for a student to resume on a different device mid-exam (explicitly deferred, not needed for v1).
- Cloud hosting — the system is self-hosted inside the university network.

## 5. User Roles & Core Needs

| Role | Core need | Primary workflows |
|---|---|---|
| **Admin** | Control and oversight of the whole system | Bulk onboarding, permissions, subject/user management, exam approval |
| **Doctor** | Run trustworthy, official assessments for their subject | Question bank, exam creation, approval submission, results/export |
| **TA** | Run lightweight formative/official quizzes for their own sections | Shared question bank, quiz creation, grading their own students |
| **Student** | Take exams fairly and securely | View available exams, take exam under lockdown, flag/review, submit |

## 6. Functional Requirements

### 6.1 Admin
- FR-1: Admin logs in with a pre-set username/password and can change their own password (not forced) after first login.
- FR-2: Admin bulk-creates/updates Students, Doctors, and TAs via Excel upload (username, password, name, role, subject/section assignments).
- FR-3: Import runs a dry-run validation pass first (shows valid vs. error rows) before committing to the database.
- FR-4: An existing username in a new upload is treated as an update, not a duplicate error.
- FR-5: Admin manages a Permissions tab — default permission set per role, with per-user overrides.
- FR-6: Admin manages Subjects, Sections, and all user records directly (outside of Excel, for corrections).
- FR-7: Admin can create/edit/delete any exam, and must approve any Doctor-authored exam before it becomes visible to students.
- FR-8: Rejecting an exam requires a reason; the Doctor can edit and resubmit.
- FR-9: Editing an already-approved exam resets it to "pending approval."
- FR-10: An exam can only be edited before its Start Time; it locks automatically once in progress.
- FR-11: Each new term, Admin re-uploads a fresh Excel roster; the previous term's data is fully replaced (acceptable since grade appeals close before the next term begins).

### 6.2 Doctor
- FR-12: Doctor maintains a private, per-subject question bank (MCQ / True-False, with Grade and Difficulty per question).
- FR-13: Questions can be added via Excel (bulk) or manually, and can include an image (via an `image_filename` column + a matching image zip upload for bulk; direct attachment for manual entry).
- FR-14: Doctor builds an exam from their own bank: selects questions, sets per-question grade, sets a fixed difficulty mix, duration, start time, and end time.
- FR-15: Exam creation is blocked with a clear error if the question bank doesn't have enough questions at a requested difficulty tier.
- FR-16: Each student receives a uniquely sampled question set matching the exam's difficulty mix.
- FR-17: Doctor-authored exams require Admin approval before publishing.
- FR-18: If a question is found flawed during/after an exam, the Doctor (or Admin) can award full credit or otherwise adjust the grade for affected students.
- FR-19: Doctor views results after an exam ends and exports them to Excel (Student Name, Student ID, Grade, with subject name + exam date as header).
- FR-20: At export time, Doctor chooses to export one exam's grades, one quiz's grades, or all exams + quizzes combined.
- FR-21: Doctor sees all students' grades (own exams and TA quizzes) grouped by section, labeled with the responsible TA's name — but never sees TAs' own submitted grading of the doctor's exam grades reversed (i.e. Doctor's own exam grades stay hidden from TAs — see FR-25).

### 6.3 Teaching Assistant (TA)
- FR-22: All TAs of a subject share one common question bank, separate from the Doctor's bank; each question is attributed to the TA who added it.
- FR-23: A TA cannot edit or delete another TA's question.
- FR-24: TA creates quizzes visible only to students in the sections they personally teach; no Admin approval required.
- FR-25: When building a quiz, the TA chooses the question source: the full shared bank, or only questions they personally added.
- FR-26: TA quiz grades are officially counted (not "practice only") and are visible to the Doctor; the Doctor's own exam grades are not visible to the TA.
- FR-27: TA sees and exports grades only for students in the sections they teach.

### 6.4 Student
- FR-28: Student sees only subjects they're enrolled in, and only exams currently within their Start/End window and matching their section (if section-targeted).
- FR-29: Exam runs inside Safe Exam Browser; the backend verifies the session via a Browser Exam Key check before granting access.
- FR-30: Exam screen is fullscreen-locked; copy/paste is disabled; student cannot exit until submission.
- FR-31: Timer starts on exam entry and cannot be paused by the student; it keeps running (wall-clock) even through a disconnection.
- FR-32: If a student disconnects and their time hasn't expired, they can log back in and resume exactly where they left off — this does not trigger auto-submit.
- FR-33: Auto-submit fires only once the exam's actual time limit runs out.
- FR-34: Student can flag any question to revisit later.
- FR-35: Student cannot submit while any question is unanswered.
- FR-36: After submission, the exam locks; the student never sees their grade or answers again on the platform.

## 7. Non-Functional Requirements

- **NFR-1 (Scale):** The system must be architected to handle ~5,000 concurrent exam-taking sessions (stress scenario), likely requiring load distribution across multiple on-prem servers.
- **NFR-2 (Hosting):** Self-hosted inside the university network — no external cloud dependency.
- **NFR-3 (Security):** Passwords are hashed on ingestion and never stored or logged in plain text; raw Excel files are not persisted after import.
- **NFR-4 (Integrity):** Exam questions/timing become immutable once the Start Time passes.
- **NFR-5 (Auditability):** All grade adjustments (question compensation, manual overrides) are attributable to the user who made them.
- **NFR-6 (Availability during exams):** A disconnection must never silently cost a student their exam progress while time remains.

## 8. Assumptions & Dependencies

- University IT will provide a Browser Exam Key for Safe Exam Browser validation. **Open question:** is it one key shared across all lab machines, or does it vary per machine/lab? This must be confirmed before final implementation (see §9).
- Safe Exam Browser is already installed and configured on lab machines — the project does not need to handle SEB deployment.
- Physical exam-taking logistics (seating, supervision, device assignment for redos) are handled by the university, not the platform.
- Grade appeal windows close before the next term begins, which is why a full data wipe-and-reload each term is acceptable.

## 9. Open Questions / Risks

| # | Question | Why it matters | Status |
|---|---|---|---|
| 1 | Is the Browser Exam Key single and fixed across all lab machines, or does it vary per machine/lab? | Determines whether the SEB middleware checks one key or a list | **Unconfirmed** — flagged for university IT |
| 2 | Realistic path to genuinely proving the 5,000-concurrent target (load testing, infra provisioning) given a self-hosted, no-cloud, single-developer graduation project | Scale claims need to be demonstrated, not just designed for | Open |

## 10. Related Documents
- Technical design (database schema + API endpoints): `exam-platform-spec.md`

## 11. Glossary
- **Doctor** — the instructor of record for a subject; owns official exams.
- **TA (Teaching Assistant)** — teaches specific sections of a subject; runs lightweight quizzes.
- **Section** — a group of students taught by a specific TA within a subject.
- **SEB** — Safe Exam Browser, the lockdown browser used during exams.
