# University Exam Platform — Full Specification

**Language:** UI in English
**Brand colors:** Primary Blue `#455B8A`, Accent Orange `#F2842F`, White `#FFFFFF`
**Stack:** Node.js + TypeScript (backend), React (frontend), PostgreSQL + Prisma ORM
**Hosting:** Self-hosted, on servers inside the university network (no external cloud). Load may be distributed across multiple servers.
**Scale target:** Must support ~5,000 concurrent students taking exams at the same instant. This is a stress-test/hypothetical ceiling rather than expected everyday load, but the architecture must be able to absorb it if it happens.

---

## 1. User Roles

| Role | Can do |
|---|---|
| **Admin** | Full system control: manage users, subjects, permissions, and exams; approve doctor exams; edit any exam before it starts. |
| **Doctor** | Owns subjects; builds a private question bank; creates official exams (Midterm/Final) that require Admin approval; views/exports results. |
| **Teaching Assistant (TA)** | Teaches specific sections of a subject; contributes to a question bank shared with the other TAs of that subject; creates quizzes for their own sections only — no approval required; views/exports grades for their own students only. |
| **Student** | Sees only their enrolled subjects and the exams/quizzes available to them; takes exams under lockdown conditions. |

---

## 2. Admin Module

### 2.1 Authentication
- Admin has an initial username/password set up outside the app (seed data).
- Admin can change their own password after logging in (not forced).

### 2.2 User Management (bulk, via Excel)
- Admin uploads an Excel file to bulk-create/update **all** users: Students, Doctors, TAs.
- Excel columns include: `username`, `password`, `full_name`, `role`, enrolled subjects (student), taught subjects (doctor), taught subjects+sections (TA).
- **No SSO / university API integration** — Excel is the only source of truth for accounts.
- **Import flow (recommended):**
  1. Admin uploads the file.
  2. System runs a **dry-run validation pass**: parses every row, flags errors (duplicate usernames, unknown subject codes, missing required fields) without writing to the DB.
  3. Admin reviews a validation report (valid rows count / error rows with reasons) before confirming.
  4. On confirm, valid rows are committed. Passwords are hashed immediately (bcrypt/argon2); the raw Excel file is not persisted after processing.
  5. If a `username` already exists, the row is treated as an **update** (not a duplicate error) — subject/section assignments are refreshed.
- **Password policy:** Students **cannot** change their password at all — it stays exactly as set in the Excel file (passwords are commonly similar/shared across students by design, which the university has accepted). Doctors/TAs/Admin **can** change their own password, but it is not forced.
- **Term refresh:** At the start of each term, the Admin re-uploads a fresh Excel file. Previous term's roster and exam data are fully wiped and replaced — this is acceptable because grade appeals/complaints close before the next term starts, so nothing of record is lost.

### 2.3 Permissions Tab
- Each role (Admin, Doctor, TA, Student) has a **default permission set** (e.g. `create_exam`, `approve_exam`, `edit_users`, `view_all_grades`, `manage_question_bank`, etc.).
- Admin can override any individual user's permissions from the default.
- Permission keys are a fixed, predefined list (not user-invented) — Admin toggles them on/off per user.

### 2.4 Subjects & Users Tab
- CRUD for Subjects.
- View/manage Doctors, Students, TAs, and general users from one place (in addition to the bulk Excel path, for ad-hoc corrections).

### 2.5 Exam Management & Approval
- Admin can create, edit, or delete any exam directly.
- Admin can define questions, grades per question, exam duration, start time, and end time on any exam.
- **Doctor-authored exams require Admin approval before they become visible to students** (Midterm/Final are treated as high-stakes). On rejection, the exam returns to the doctor with a mandatory reason/comment; the doctor edits and resubmits (never deleted).
- Any edit to an already-approved exam resets its status back to "pending approval."
- **TA-authored quizzes do NOT require Admin approval** — they publish directly.
- An exam can only be edited **before** its Start Time. Once in progress, questions/timing are locked (they're already distributed to students).

---

## 3. Doctor Module

### 3.1 Question Bank (private per doctor, per subject)
- Each question has: `Question text`, `Answers` (options), `Correct Answer`, `Grade`, `Difficulty` (Easy / Medium / Hard — 3 fixed tiers).
- v1 question types: **Multiple Choice** and **True/False** only.
- Questions can include an **image**. Since images can't be embedded in Excel, the import flow is:
  1. Excel includes an `image_filename` column.
  2. Doctor uploads a matching folder/zip of image files alongside the Excel file.
  3. The system auto-links each question to its image by matching filename during import.
  4. For manually-added questions, the doctor attaches the image directly in the form.
- Questions can be added via Excel bulk upload or manually (Add/Edit/Delete), both before and after any Excel import.
- This bank is **fully separate** from the shared TA bank of the same subject.

### 3.2 Exam Creation
- Doctor builds an exam by selecting from their own question bank.
- Doctor sets: which questions, grade per question, difficulty mix (e.g. 5 easy / 10 medium / 5 hard — a **fixed mix per exam**, not uniform, not per-student adaptive), exam duration, Start Time, End Time.
- **Validation rule:** if the question bank doesn't have enough questions at a given difficulty tier to satisfy the exam's requested mix, exam creation is blocked with a clear error (never silently repeats the same question across many students).
- **Unique exam per student:** each student's exam is generated by randomly sampling from the doctor's question pool according to the fixed difficulty mix, so no two students get an identical question order/set where avoidable.
- Requires Admin approval to publish (see §2.5).

### 3.3 Problem Question Handling
- If a doctor discovers a flawed question during/after an exam, they (or the Admin) can give affected students full credit for that question, or otherwise manually adjust the grade.
- If a student needs a full redo (e.g., their session broke): treated as a completely ordinary new exam — there is **no special "compensatory exam" feature**. The university itself organizes students' access to devices for a redo; the system doesn't need extra logic for this.

### 3.4 Results
- After an exam ends, doctor can view student results/grades.
- Doctor can export results to Excel with **Student Name, Student ID, Grade**, plus the subject name and exam date/time as a header.
- **Export choices offered:** export a single exam's grades, a single quiz's grades, or all exams + quizzes combined — doctor picks from these options at export time.

### 3.5 Grade Visibility (Doctor side)
- Doctor sees grades for **all** students across all TAs of the subject, in a dedicated part of the UI, grouped by section — each group clearly labeled with the responsible TA's name.
- Doctor sees TA quiz grades.
- Doctor's own exam grades are **not** visible to TAs.
- All grades (doctor exams and TA quizzes alike) are **officially counted** — nothing here is "practice only."

---

## 4. Teaching Assistant (TA) Module

### 4.1 Question Bank (shared among all TAs of a subject)
- All TAs teaching the same subject share **one common bank** (separate from the doctor's bank).
- Each question shows who added it (attribution).
- A TA **cannot** edit or delete a question added by a different TA — only their own.
- Same question fields/import mechanics as the Doctor bank (§3.1), including the image_filename + zip approach.

### 4.2 Quizzes
- A TA creates and runs quizzes for the students in the sections **they personally teach** — not the whole subject.
- Example: TA "Ahmed" teaches Sections A, B, C of Software Engineering → his quiz is visible only to students in A, B, C.
- When building a quiz, the TA chooses the question source: the **full shared bank**, or **only questions they personally added**.
- No Admin approval needed — quizzes publish directly.
- Uses the same underlying exam mechanism as a doctor's exam (just a `type` flag: `exam` vs `quiz`).

### 4.3 Grade Visibility (TA side)
- A TA sees grades only for students in the sections/groups **they** teach — never other TAs' students.
- A TA can export their own students' grades to Excel.

---

## 5. Student Module

### 5.1 Subjects & Exams
- After login, student sees only the subjects they're enrolled in (from the Admin's Excel data).
- Student sees an exam/quiz only if:
  1. They're enrolled in that subject.
  2. The exam is currently within its Start/End window.
  3. If the exam targets specific sections, the student belongs to one of those sections.

### 5.2 Taking an Exam
- **Lockdown:** exam runs inside **Safe Exam Browser (SEB)**, which is already installed on the lab machines. The system verifies the request came from a real SEB session via the **Browser Exam Key** header check (see §7).
- Screen goes fullscreen and is locked; copy/paste is disabled; student cannot exit until submission.
- Timer starts the moment the exam begins and **cannot be paused** by the student.
- **Timer keeps running (wall-clock) even if the student disconnects.** If the student's time hasn't run out yet, they can log back in and resume exactly where they left off — this is **not** treated as an auto-submit event. Auto-submit only fires once the exam's actual time limit expires.
- Disconnections are still tracked via a heartbeat/ping (for monitoring/status visibility), but no longer force submission.
- Student can **flag** a question to revisit later.
- Student **cannot submit** while any question is unanswered — every question must have an answer first.
- After submission, the exam locks; the student never sees their grade or answers again through the platform.

---

## 6. Security: Safe Exam Browser Integration

SEB is already installed on lab machines, so there's no need to generate `.seb` config files for v1. The lightweight approach:

1. When a student opens the exam through SEB (not a normal browser), SEB automatically attaches an `X-SafeExamBrowser-RequestHash` header to every request — a hash of (page URL + the machine's configured **Browser Exam Key**).
2. The backend, knowing the expected Browser Exam Key, computes the same hash for the incoming request URL and compares it to the header.
3. Match → genuine SEB session, allow entry to the exam.
4. Missing header or mismatch → reject with "Please open this exam from Safe Exam Browser."

> ⚠️ **Assumption to confirm:** this design currently assumes **one single, fixed Browser Exam Key shared across all lab machines**. This must be confirmed with university IT before final implementation — if different machines/labs actually use different keys, the backend needs to accept a list of valid keys instead of a single constant.

---

## 7. Database Design

Below is the core relational schema (PostgreSQL, via Prisma). Field types are illustrative; adjust precision as needed.

### 7.1 Identity & Structure

**User**
| Field | Type | Notes |
|---|---|---|
| id | uuid PK | |
| username | string, unique | |
| password_hash | string | bcrypt/argon2; never store raw |
| full_name | string | |
| role | enum(admin, doctor, ta, student) | |
| can_change_password | boolean | false for students, true otherwise |
| is_active | boolean | default true |
| created_at / updated_at | timestamp | |

**Subject**
| Field | Type | Notes |
|---|---|---|
| id | uuid PK | |
| code | string, unique | e.g. `CS301` |
| name | string | |

**Enrollment** (student ↔ subject, many-to-many)
| student_id | subject_id |

**DoctorAssignment** (doctor ↔ subject they own/teach)
| doctor_id | subject_id |

**Section** (a TA's teaching group within a subject)
| Field | Type |
|---|---|
| id | uuid PK |
| subject_id | FK → Subject |
| ta_id | FK → User (role=ta) |
| name | string, e.g. "Section A" |

**SectionMembership** (student ↔ section, many-to-many)
| student_id | section_id |

### 7.2 Question Banks

**Question**
| Field | Type | Notes |
|---|---|---|
| id | uuid PK | |
| subject_id | FK → Subject | |
| owner_type | enum(doctor, ta_shared) | doctor-private vs. TA shared-per-subject |
| doctor_id | FK → User, nullable | set when owner_type = doctor |
| added_by_ta_id | FK → User, nullable | set when owner_type = ta_shared (attribution; also the only TA allowed to edit/delete it) |
| question_type | enum(mcq, true_false) | |
| text | text | |
| options | jsonb | answer choices |
| correct_answer | string | |
| grade | numeric | |
| difficulty | enum(easy, medium, hard) | |
| image_url | string, nullable | |
| created_at | timestamp | |

### 7.3 Exams

**Exam**
| Field | Type | Notes |
|---|---|---|
| id | uuid PK | |
| subject_id | FK → Subject | |
| type | enum(doctor_exam, ta_quiz) | |
| created_by | FK → User | |
| status | enum(draft, pending_approval, approved, rejected, locked, closed) | |
| rejection_reason | text, nullable | |
| approved_by | FK → User, nullable | |
| start_time / end_time | timestamp | |
| duration_minutes | int | |
| difficulty_mix | jsonb | e.g. `{"easy": 5, "medium": 10, "hard": 5}` |
| target_scope | enum(subject, sections, student_list) | how the exam is targeted |
| created_at / updated_at | timestamp | |

**ExamTargetSection** — used when `target_scope = sections`
| exam_id | section_id |

**ExamTargetStudent** — used when `target_scope = student_list` (e.g. an ad-hoc redo exam for specific students)
| exam_id | student_id |

**StudentExam** (one per student per exam — the generated, unique instance)
| Field | Type | Notes |
|---|---|---|
| id | uuid PK | |
| exam_id | FK → Exam | |
| student_id | FK → User | |
| status | enum(not_started, in_progress, submitted, auto_submitted) | |
| started_at | timestamp, nullable | |
| submitted_at | timestamp, nullable | |
| total_grade | numeric, nullable | computed on submission |
| last_heartbeat_at | timestamp, nullable | for disconnect monitoring only — does not pause the timer |

**StudentExamQuestion** (the specific, randomly-sampled question set given to this student)
| Field | Type | Notes |
|---|---|---|
| id | uuid PK | |
| student_exam_id | FK → StudentExam | |
| question_id | FK → Question | |
| selected_answer | string, nullable | |
| is_flagged | boolean | default false |
| grade_awarded | numeric, nullable | |
| credit_adjustment_reason | text, nullable | set if this question was voided/compensated |

### 7.4 Access Control

**Permission** (default matrix)
| role | permission_key | allowed |

**UserPermissionOverride**
| user_id | permission_key | allowed |

### 7.5 Import Auditing

**ExcelImportLog**
| Field | Type |
|---|---|
| id | uuid PK |
| import_type | enum(users, questions) |
| uploaded_by | FK → User |
| filename | string |
| total_rows | int |
| imported_count | int |
| error_count | int |
| error_report | jsonb |
| created_at | timestamp |

---

## 8. API Design (REST)

Base path: `/api/v1`

### Auth
- `POST /auth/login` — username + password → JWT/session
- `POST /auth/change-password` — self-service (blocked for students)

### Admin — Users & Subjects
- `POST /admin/users/import/dry-run` — upload Excel, return validation report without committing
- `POST /admin/users/import/commit` — commit a previously validated import
- `GET /admin/users` / `PATCH /admin/users/:id` — ad-hoc management
- `POST /admin/subjects` / `GET /admin/subjects` / `PATCH /admin/subjects/:id`
- `POST /admin/sections` — create a section and assign a TA

### Admin — Permissions
- `GET /admin/permissions/defaults`
- `PATCH /admin/permissions/user/:id` — override a user's permissions

### Admin — Exam Oversight
- `GET /admin/exams?status=pending_approval`
- `POST /admin/exams/:id/approve`
- `POST /admin/exams/:id/reject` — body: `{ reason }`
- `PATCH /admin/exams/:id` — allowed only if `now < start_time`

### Question Bank (Doctor & TA — scope enforced server-side by role/subject/ownership)
- `GET /question-bank?subject_id=`
- `POST /question-bank` — supports `owner_type` auto-set from caller's role
- `PATCH /question-bank/:id` / `DELETE /question-bank/:id` — server checks ownership (own questions only for TAs)
- `POST /question-bank/import/dry-run` — Excel + optional image zip
- `POST /question-bank/import/commit`

### Exams
- `POST /exams` — create (doctor → status `pending_approval`; TA quiz → status `approved` immediately)
- `GET /exams/:id`
- `PATCH /exams/:id` — locked once `now >= start_time`
- `DELETE /exams/:id`
- `POST /exams/:id/compensate` — body: `{ question_id, adjustment_type, adjustment_value, student_ids? }`

### Student Exam-Taking
- `GET /student/exams` — available exams for the logged-in student
- `POST /student/exams/:examId/start` — generates the StudentExam + sampled question set on first call; returns existing state if resuming
- `PATCH /student/exams/:examId/answer` — body: `{ question_id, selected_answer }`
- `PATCH /student/exams/:examId/flag` — body: `{ question_id, is_flagged }`
- `POST /student/exams/:examId/heartbeat` — periodic ping (updates `last_heartbeat_at` only, never pauses timer)
- `POST /student/exams/:examId/submit` — rejected with 409 if any question is unanswered

### Results & Export
- `GET /doctor/exams/:examId/results`
- `GET /doctor/results/export?scope=exam:{id}|quiz:{id}|all`
- `GET /ta/results/export` — own students only

### SEB Verification (middleware, not a public endpoint)
- Applied to all `/student/exams/:examId/*` routes: validates `X-SafeExamBrowser-RequestHash` against the configured Browser Exam Key(s) before allowing access.

---

## 9. Open Items To Confirm

- **Browser Exam Key:** confirm with university IT whether one fixed key is shared across all lab machines, or whether it varies per machine/lab (affects §6 and the SEB middleware).
