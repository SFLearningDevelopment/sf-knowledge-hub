# SF Knowledge Hub — Doc 01: Course Creation Kickoff Prompt

Paste this into a **new Claude chat (Opus 4.8, High)** to begin a new course.
Fill the bracketed placeholders before sending.

---

## Kickoff Prompt (copy from here)

```
You are helping me build a new course on the SF Knowledge Hub — SourceFuse's internal
certification and skills platform. Read the context below before we start.

PLATFORM
- Hosting: GitHub Pages, repo root `sflearningdevelopment.github.io/`.
- Backend: Firebase project `sfcertification-grafana`, Firestore.
- Auth: Google SSO with `@sourcefuse.com` enforcement; email/password fallback.

GOLDEN RULE (non-negotiable)
- Before modifying or fixing ANY file, ASK ME for the latest deployed version of that
  file and wait for me to paste it. Never edit from a session copy or from memory.

FIRESTORE COLLECTIONS
- Progress (dual-write): legacy `progress/{uid}` + v2 `courseProgress/{uid}`.
- Exam results: `courseExamResults/{email}/courses/{courseId}/attempts/{id}`.
- Course catalog: `courses` (each course = one doc; its `modules` array `file`
  entries must match the moduleKeys in `courseProgress` exactly).
- Exam catalog: `exams` (dropdowns list every doc; `linkedCourseId` must equal the
  course's catalog doc ID / the courseId used in courseExamResults).

THIS COURSE
- Course title: [e.g. "{Tool} Essentials"]
- Certification title: [e.g. "SF Certified {Tool} Practitioner"]
- Folder: [e.g. `tool-name/` under repo root]
- courseId / catalog doc ID: [e.g. `xxxxxxxxxxxxxxxxxxxx`]  (or: "assign on creation")
- Template type: [TYPE 1 practice/Claude-for-PM  OR  TYPE 2 essentials/QuickSight]
- Modules: [e.g. 5 modules — list titles]
- Question bank: [e.g. 45 questions, `window.TOOL_QUESTION_BANK`]
- courseProgress moduleKeys: [e.g. `tool_m1`…`tool_m5`]

CONVENTIONS
- Cache-busting every iteration: `<!-- build: YYYYMMDDNN -->` on line 2 of HTML,
  `/* build: … */` in JS; `?v=BUILD` on internal cross-page links, never on
  `→course-map` links or admin `url` fields.
- Firestore rules and any plain-text config delivered as `.txt`.
- Content addresses the learner as "you"; no meta-commentary to me inside deliverables.
- Avoid human-body metaphors ("spine", "backbone", "heart of"); use "through-line",
  "central thread", "foundation".

TEMPLATES
- I will paste the TYPE 1 / TYPE 2 template file(s) in my next message. Templates
  cannot be auto-loaded across chats, so wait for them before generating anything.

Confirm you've read this, then ask me for (a) the template file(s) and (b) any deployed
file you need. Do not generate course content until I provide the templates.
```

---

## Notes for the creator
- Templates **must** be re-pasted in every new course chat — they don't carry over.
- If courseId is "assign on creation," decide it before wiring the `courses` catalog doc,
  since the folder, `modules[].file`, `courseProgress` keys, and `exams.linkedCourseId`
  all reference it.
