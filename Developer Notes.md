# SF Certifications Platform — Exam Developer Notes

**Project:** `sf-knowledge-hub` (sflearningdevelopment.github.io/sf-knowledge-hub/)
**Firebase:** `sfcertification-grafana`
**Last updated:** 2026-03-24

---

## Overview

This document describes everything a developer needs to know when building a new exam page
(e.g. Kubernetes, AWS, Databricks) that integrates with the SF Certifications Hub (`index.html`)
and Admin Dashboard (`sf-certs-hub.html`).

The hub manages the full exam lifecycle — slot booking, proctor assignment, identity verification,
and admission. Once admitted, the candidate clicks "Begin Exam" and is opened into your exam page
in a new tab. **Your exam page is responsible for Writes 1 and 2 only (score writes). All
platform-state writes are handled by index.html.**

---

## 1. URL Parameters — What index.html Passes to Your Exam Page

When `index.html` opens your exam page, it appends four URL parameters:

```
https://yourexam.html?slotId=X&examId=Y&courseId=Z&duration=45
```

| Param | Value | Used for |
|---|---|---|
| `slotId` | `examSlots` doc ID | Write 5, tab-switch flags, session doc |
| `examId` | `exams` collection doc ID | Write 2 key in `examResults` |
| `courseId` | `courses` collection doc ID | Write 4 progress unlock |
| `duration` | Minutes (e.g. `45`) | Timer |

Read all four at page load after auth:

```javascript
const _urlParams  = new URLSearchParams(window.location.search);
slotId       = _urlParams.get('slotId')   || null;
examDocId    = _urlParams.get('examId')   || null;
examCourseId = _urlParams.get('courseId') || null;
const duration = parseInt(_urlParams.get('duration') || '45', 10);
```

**There is no need to query Firestore for the exam or course ID.** The old pattern of
`findGrafanaExamId()` / `findGrafanaCourseId()` is removed. Do not implement title-based
lookups in new exam pages.

---

## 2. Required Module-Level State Variables

```javascript
let slotId         = null;   // from URL param
let examDocId      = null;   // from URL param — key in examResults
let examCourseId   = null;   // from URL param — for progress unlock
let _timedOut      = false;  // set true ONLY when timer fires submitExam
let sessionDocId   = null;   // examSessions doc ID — created at startExam
let _tabSwitchCount = 0;     // running count of tab switches
```

---

## 3. Session Document — Create at Exam Start

Create the `examSessions` doc when the candidate starts (not at submit):

```javascript
// At startExam() — BEFORE renderExam() and startTimer()
const sessRef = await addDoc(collection(db, 'examSessions'), {
  candidateUid:     currentUser.uid,
  candidateEmail:   currentUser.email,
  candidateName:    currentUser.displayName || currentUser.email,
  examId:           examDocId || 'unknown',
  slotId:           slotId || null,
  startedAt:        serverTimestamp(),
  status:           'active',
  flags:            [],
  questionProgress: 0,
  totalQuestions:   <your total question count>
});
sessionDocId = sessRef.id;
```

Then attach the shared integrity listeners (see Section 6).

---

## 4. Timer — Set _timedOut Before submitExam

```javascript
if (timeLeft <= 0) {
  clearInterval(timerInt);
  _timedOut = true;          // MUST be set before submitExam()
  showToast('Time is up! Submitting…', 'error');
  submitExam();
}
```

`_timedOut` propagates to the localStorage signal. `index.html` uses it to write
`status: 'abandoned'` vs `status: 'completed'` to `examSlots`. The Kubernetes "Failed"
instead of "Abandoned" bug was caused by not setting this flag.

---

## 5. The Two Exam-Specific Writes at Submit

**Your exam page only handles Writes 1 and 2.** Writes 3, 4, 5 are handled by `index.html`
after receiving the localStorage signal.

### Write 1 — `results/{uid}` (Legacy, backward-compatible)
```javascript
await setDoc(doc(db, 'results', currentUser.uid), {
  uid, email, lastScore, lastPassed, attemptCount,
  lastAttemptAt: serverTimestamp(),
  [`attempt_${n}_score`]: score,
  [`attempt_${n}_passed`]: passed,
  [`attempt_${n}_time`]: timeTaken,
}, { merge: true });
```

### Write 2 — `examResults/{uid}` (Hub Test Results report — REQUIRED)
```javascript
await setDoc(doc(db, 'examResults', currentUser.uid), {
  learnerEmail: candidateEmail,
  learnerName:  candidateName,
  [examDocId]: {                  // key is the examId URL param
    attemptCount: n,
    bestScore, bestPassed,
    lastScore, lastPassed,
    lastAttemptAt: serverTimestamp(),
    attempts: [...previousAttempts, { at, score, passed, timeTaken }]
  }
}, { merge: true });
```

> **Critical:** Use `examDocId` (from URL param) as the key. Never hardcode a string.

---

## 6. Shared Exam Integrity Module — Copy Verbatim

The block between `/* ══ SHARED EXAM INTEGRITY MODULE ══ */` and
`/* ══ END SHARED EXAM INTEGRITY MODULE ══ */` in `grafana-evaluation.html` must be
copied verbatim into every new exam page. Do not rename functions.

It provides:
- `_logExamEvent(type, extra)` — writes to `examSessions.flags[]` and `auditLog`
- `_onVisibilityChange()` — tab switch detection
- `_onBeforeUnload(e)` — exit warning + logging
- `_removeExamListeners()` — cleanup at submit

Attach listeners at `startExam()` after creating the session doc:
```javascript
document.addEventListener('visibilitychange', _onVisibilityChange);
window.addEventListener('beforeunload', _onBeforeUnload);
```

Remove at top of `submitExam()`:
```javascript
_removeExamListeners();
```

---

## 7. The localStorage Signal — Triggers index.html Writes 3/4/5

After Writes 1 and 2 succeed, send this signal:

```javascript
localStorage.setItem('sf_exam_done', JSON.stringify({
  slotId,
  examId:         examDocId || null,
  courseId:       examCourseId || null,
  sessionDocId:   sessionDocId || null,
  candidateUid:   currentUser.uid,
  candidateEmail: currentUser.email,
  candidateName:  currentUser.displayName || currentUser.email,
  score,
  passed,
  timeTaken,
  timedOut:         _timedOut,          // true = abandoned, false = completed
  tabSwitchCount:   _tabSwitchCount,
  ts: Date.now()
}));
```

`index.html` receives this via `window.addEventListener('storage')` and executes:

| Write | Collection | Trigger |
|---|---|---|
| 3 | `examSessions/{sessionDocId}` | Always — updates flags + final score |
| 4 | `progress/{uid}` | Only if `passed === true` and `courseId` set |
| 5 | `examSlots/{slotId}` | Always — `completed` or `abandoned` based on `timedOut` |

---

## 8. Back to Hub and Retry Links

All "Back to Hub" links and the Retry Exam button must point to the hub:

```
https://sflearningdevelopment.github.io/sf-knowledge-hub/index.html
```

**Never use `location.reload()` for Retry** — it reloads the exam page instead of returning
to the hub. Use `window.location.href = HUB_URL` instead.

---

## 9. Checklist for a New Exam Page

- [ ] Read `slotId`, `examId`, `courseId`, `duration` from URL params after auth
- [ ] Create `examSessions` doc at `startExam()`, store `sessionDocId`
- [ ] Copy Shared Exam Integrity Module verbatim
- [ ] Attach `visibilitychange` + `beforeunload` listeners at `startExam()`
- [ ] Set `_timedOut = true` in timer **before** calling `submitExam()`
- [ ] Call `_removeExamListeners()` at top of `submitExam()`
- [ ] Write 1: `results/{uid}` (legacy)
- [ ] Write 2: `examResults/{uid}` keyed by `examDocId` URL param
- [ ] Send `localStorage.setItem('sf_exam_done', ...)` with full payload
- [ ] "Back to Hub" link → `sf-knowledge-hub/index.html`
- [ ] "Retry Exam" → `window.location.href = HUB_URL` (NOT `location.reload()`)
- [ ] Console prefix: `[your-exam-eval]`

---

## 10. Firestore Collections Reference

| Collection | Written by | Read by | Notes |
|---|---|---|---|
| `examSlots/{slotId}` | `index.html` Write 5 | `sf-certs-hub.html` | `completed` / `abandoned` terminal statuses |
| `examResults/{uid}` | Exam page Write 2 | `sf-certs-hub.html` Reports | Key = `examId` URL param |
| `examSessions/{id}` | Exam page (create) + `index.html` (update) | Reports, Live Monitor | Create at start, update via hub signal |
| `results/{uid}` | Exam page Write 1 | Legacy only | Not read by hub |
| `progress/{uid}` | `index.html` Write 4 | `index.html` course unlock | Only on pass |
| `auditLog/{id}` | Exam page + `index.html` | Audit Log tab | Permanent, admin-only read |

---

## 11. Debugging Tips

- Log `slotId`, `examDocId`, `examCourseId` right after auth — if any are null, related writes will silently skip
- Check `sessionDocId` after session doc creation
- Use Firebase SDK `12.10.0` to avoid mismatch errors
- Confirm Firestore rules v1.8+ allow `isSF()` writes to `examSessions`, `examResults`, `examSlots`, `auditLog`, `progress`
- "Abandoned" not showing? Check `_timedOut = true` is set in the timer before `submitExam()` is called
- Results not in hub Test Results? Check `examDocId` matches the doc ID in the `exams` Firestore collection
