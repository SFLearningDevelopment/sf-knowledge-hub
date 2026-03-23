# SF Certifications Platform — Exam Developer Notes

**Project:** `sf-knowledge-hub` (sflearningdevelopment.github.io/sf-knowledge-hub/)
**Firebase:** `sfcertification-grafana`
**Reference implementation:** `grafana-evaluation.html`
**Last updated:** 2026-03-23

---

## Overview

This document describes everything a developer needs to know when building a **new exam page** (e.g. Kubernetes, AWS, Databricks) that integrates with the SF Certifications Hub (`index.html`) and Admin Dashboard (`sf-certs-hub.html`).

The hub manages the full exam lifecycle — slot booking, proctor assignment, identity verification, and admission. Once admitted, the candidate clicks "Begin Exam" and is opened into your exam page in a new tab. **Your exam page is responsible for everything from that point onward.**

---

## 1. How Your Exam Page Receives Context

When `index.html` opens your exam page, it appends a `slotId` query parameter to the URL:

```
https://yourexam.html?slotId=<examSlotDocId>
```

Read this at page load (after auth):

```javascript
const slotId = new URLSearchParams(window.location.search).get('slotId') || null;
```

**`slotId` is the Firestore document ID in `examSlots/{slotId}`.** It links your exam page to the candidate's booking, proctor assignment, and proctoring snapshots. It is the key that connects all systems together.

If `slotId` is null, the exam was opened directly (not through the hub admission flow). All Firestore writes guarded by `if (slotId)` will simply be skipped — this is safe.

---

## 2. Required Module-Level State Variables

```javascript
let slotId        = null;   // from URL param — set after auth
let _timedOut     = false;  // set to true ONLY when timer fires submitExam
let sessionDocId  = null;   // examSessions doc ID — created at startExam, updated at submitExam
let _tabSwitchCount = 0;    // running count of tab switches during exam
```

You also need to resolve the exam's Firestore doc ID (used as the key in `examResults`):

```javascript
let myExamId = null; // resolved from 'exams' collection by title match
```

See `findGrafanaExamId()` in `grafana-evaluation.html` for the title-query pattern. Each exam page implements its own version of this function.

---

## 3. Session Document — Create at Exam Start, Not at Submit

**Critical:** Create the `examSessions` document when the candidate **starts** the exam (clicks the Start button), not when they submit. This allows flags (tab switches, exit attempts, face-away events) to be appended throughout the exam.

```javascript
// At startExam() — BEFORE renderExam() and startTimer()
const sessRef = await addDoc(collection(db, 'examSessions'), {
  candidateUid:     currentUser.uid,
  candidateEmail:   currentUser.email,
  candidateName:    currentUser.displayName || currentUser.email,
  examId:           myExamId || 'unknown',   // your resolved exam doc ID
  slotId:           slotId || null,
  startedAt:        serverTimestamp(),
  status:           'active',
  flags:            [],
  questionProgress: 0,
  totalQuestions:   <your total question count>
});
sessionDocId = sessRef.id;
```

**Why this matters:** `sf-certs-hub.html` reads `examSessions` in real-time during Live Monitor and Reports. If the doc is only created at submit, the proctor sees nothing during the exam and tab-switch counts are lost.

---

## 4. Tab Switch and Exit Detection

Attach these listeners immediately after creating the session doc in `startExam()`:

```javascript
document.addEventListener('visibilitychange', _onVisibilityChange);
window.addEventListener('beforeunload', _onBeforeUnload);
```

**`_onVisibilityChange`:** Detects when the candidate switches away from the exam tab.

```javascript
function _onVisibilityChange() {
  if (!examStarted || examSubmitted) return;
  if (document.hidden) {
    _tabSwitchCount++;
    _logExamEvent('tab_switch', {
      count: _tabSwitchCount,
      message: 'Candidate switched away from exam tab (switch #' + _tabSwitchCount + ')'
    });
  } else {
    _logExamEvent('tab_switch_return', {
      count: _tabSwitchCount,
      message: 'Candidate returned to exam tab after switch #' + _tabSwitchCount
    });
  }
}
```

**`_onBeforeUnload`:** Warns the candidate and logs the attempt if they try to close or navigate away.

```javascript
function _onBeforeUnload(e) {
  if (!examStarted || examSubmitted) return;
  const msg = 'Your exam is in progress. Leaving this page may result in your attempt being marked as abandoned.';
  e.preventDefault();
  e.returnValue = msg;
  _logExamEvent('exit_attempt', {
    message: 'Candidate attempted to close/navigate away (tab switches so far: ' + _tabSwitchCount + ')'
  });
  return msg;
}
```

**Remove both listeners at the top of `submitExam()`:**

```javascript
document.removeEventListener('visibilitychange', _onVisibilityChange);
window.removeEventListener('beforeunload', _onBeforeUnload);
```

---

## 5. Logging Events — `_logExamEvent(type, extra)`

All integrity events must be written to **two places simultaneously**:

1. **`examSessions/{sessionDocId}.flags[]`** — read by proctor Live Monitor and Admin Reports
2. **`auditLog`** — permanent record readable only by the super-admin

```javascript
async function _logExamEvent(type, extra = {}) {
  const entry = { type, at: new Date().toISOString(), ts: Date.now(), ...extra };

  // 1. Append to examSessions flags array
  if (sessionDocId) {
    try {
      const ref  = doc(db, 'examSessions', sessionDocId);
      const snap = await getDoc(ref);
      const flags = (snap.exists() ? snap.data().flags : []) || [];
      flags.push(entry);
      await setDoc(ref, { flags, lastFlagAt: serverTimestamp() }, { merge: true });
    } catch(e) { console.warn('flag write failed:', e.message); }
  }

  // 2. Write to auditLog
  try {
    await addDoc(collection(db, 'auditLog'), {
      action:       type,
      actorEmail:   currentUser?.email || 'unknown',
      actorRole:    'student',
      targetEmail:  currentUser?.email || 'unknown',
      slotId:       slotId || null,
      examId:       myExamId || null,
      sessionDocId: sessionDocId || null,
      metadata:     extra.message || type,
      timestamp:    serverTimestamp()
    });
  } catch(e) { console.warn('auditLog write failed:', e.message); }
}
```

**Supported event types (used by `sf-certs-hub.html` reports):**

| `type` | When | Reported as |
|---|---|---|
| `tab_switch` | Candidate hides exam tab | Tab count in Reports |
| `tab_switch_return` | Candidate returns to exam tab | Duration calculable |
| `exit_attempt` | Candidate tries to close/navigate | Flag in Reports |
| `face_away` | Face not detected for >N seconds | Face-away count in Reports |

---

## 6. The Five Firestore Writes at Submit

Execute these in order inside `submitExam()`. Each write is wrapped in its own try/catch so one failure doesn't block the others.

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
Kept for backward compatibility. `sf-certs-hub.html` does NOT read this collection for its reports — it reads `examResults` (Write 2).

### Write 2 — `examResults/{uid}` (Hub-compatible — REQUIRED for Test Results report)
```javascript
await setDoc(doc(db, 'examResults', currentUser.uid), {
  learnerEmail: candidateEmail,   // identity fields for name lookup
  learnerName:  candidateName,
  [myExamId]: {                   // keyed by exam doc ID from 'exams' collection
    attemptCount: n,
    bestScore, bestPassed,
    lastScore, lastPassed,
    lastAttemptAt: serverTimestamp(),
    attempts: [...previousAttempts, { at, score, passed, timeTaken }]
  }
}, { merge: true });
```

> **Important:** The key inside `examResults/{uid}` must be the Firestore document ID from the `exams` collection — not a hardcoded string. Resolve it at init time using a title query (see `findGrafanaExamId()` pattern). If this ID is wrong, the candidate's result will never appear in the admin Test Results report.

### Write 3 — `examSessions/{sessionDocId}` (Update existing doc)
```javascript
// Update the doc created at startExam — preserves all flags
if (sessionDocId) {
  await setDoc(doc(db, 'examSessions', sessionDocId), {
    score, passed, timeTaken,
    tabSwitchCount: _tabSwitchCount,
    completedAt:    serverTimestamp(),
    status:         passed ? 'passed' : 'failed',
    completionType: _timedOut ? 'timeout' : 'submitted',
    slotId:         slotId || null,
  }, { merge: true });
} else {
  // Fallback if page was refreshed mid-exam and sessionDocId was lost
  await addDoc(collection(db, 'examSessions'), { ...allFields, flags: [] });
}
```

> **Do NOT create a new doc here.** Updating the existing doc preserves all `flags[]` entries written during the exam (tab switches, exit attempts, face events). Creating a new doc loses all of them.

### Write 4 — `progress/{uid}` (Course completion — only if passed)
```javascript
if (passed && myCourseId) {
  await setDoc(doc(db, 'progress', currentUser.uid), {
    [`course_${myCourseId}`]: true,
    updatedAt: serverTimestamp()
  }, { merge: true });
}
```
Unlocks the course completion badge on the candidate's hub. Only write if the candidate passed.

### Write 5 — `examSlots/{slotId}` (Status update — REQUIRED to clear "Exam In Progress")
```javascript
if (slotId) {
  await setDoc(doc(db, 'examSlots', slotId), {
    status:         _timedOut ? 'abandoned' : 'completed',
    completionType: _timedOut ? 'timeout'   : 'submitted',
    completedAt:    serverTimestamp(),
    finalScore:     score,
    finalPassed:    passed,
  }, { merge: true });
}
```

> **Critical:** Without this write, the proctor dashboard will show "Exam In Progress" indefinitely for past slots. `'completed'` and `'abandoned'` are the two valid terminal statuses. `_timedOut` must be set to `true` by the timer before calling `submitExam()`.

---

## 7. Timer — Mark Timeout Before Calling submitExam

```javascript
if (timeLeft <= 0) {
  clearInterval(timerInt);
  _timedOut = true;             // MUST be set before submitExam()
  showToast('Time is up! Submitting…', 'error');
  submitExam();
}
```

When the candidate clicks the Submit button manually, `_timedOut` remains `false`. This distinction propagates to Write 3 (`completionType: 'submitted'`) and Write 5 (`status: 'completed'`).

---

## 8. Hub Tab Signal — localStorage

After all five writes succeed, signal `index.html` to clear its navigation lock:

```javascript
try {
  localStorage.setItem('sf_exam_done', JSON.stringify({
    slotId,
    status: _timedOut ? 'abandoned' : 'completed',
    examId: myExamId || null,
    ts:     Date.now()
  }));
} catch(e) { console.warn('localStorage signal failed:', e.message); }
```

`index.html` listens for this via `window.addEventListener('storage', ...)`. When received, it:
- Removes the `beforeunload` warning from the hub tab
- Writes an `exam_completed_signal` entry to `auditLog`
- Calls `loadStudentExams()` to refresh the candidate's exam card

---

## 9. Abandoned Exam Detection — How `index.html` Handles It

`index.html` does **not** run its own timeout timer for abandon detection. Instead:

- Write 5 marks the slot `abandoned` when `_timedOut = true` in the exam page
- The exam page's own timer is the authoritative source — it sets `_timedOut` before calling `submitExam()`

If the candidate closes the exam tab without submitting and without the timer firing (e.g. browser crash), the slot remains `inProgress`. This is an edge case with no clean resolution in a static GitHub Pages architecture without a Cloud Function.

---

## 10. Checklist for a New Exam Page

- [ ] Read `slotId` from URL params after auth
- [ ] Resolve exam doc ID from `exams` collection by title at init (`findMyExamId()`)
- [ ] Resolve course doc ID from `courses` collection by title at init (`findMyCourseId()`)
- [ ] Create `examSessions` doc at `startExam()`, store `sessionDocId`
- [ ] Attach `visibilitychange` and `beforeunload` listeners at `startExam()`
- [ ] Set `_timedOut = true` in timer **before** calling `submitExam()`
- [ ] Call `_removeExamListeners()` at top of `submitExam()`
- [ ] Write 1: `results/{uid}` — legacy
- [ ] Write 2: `examResults/{uid}` — keyed by exam doc ID (NOT a hardcoded string)
- [ ] Write 3: update `examSessions/{sessionDocId}` with final result
- [ ] Write 4: `progress/{uid}` — only if passed
- [ ] Write 5: `examSlots/{slotId}` — `completed` or `abandoned`
- [ ] Signal hub: `localStorage.setItem('sf_exam_done', ...)`
- [ ] Console logs: prefix with `[your-exam-eval]` for easier debugging

---

## 11. Firestore Collections Reference

| Collection | Written by | Read by | Notes |
|---|---|---|---|
| `examSlots/{slotId}` | `index.html`, exam page (Write 5) | `sf-certs-hub.html` proctor/admin | Terminal statuses: `completed`, `abandoned` |
| `examResults/{uid}` | exam page (Write 2) | `sf-certs-hub.html` Test Results | Key is exam doc ID from `exams` collection |
| `examSessions/{id}` | exam page (Write 3) | `sf-certs-hub.html` Reports, Live Monitor | Create at start, update at end |
| `results/{uid}` | exam page (Write 1) | Not read by hub | Legacy only |
| `progress/{uid}` | exam page (Write 4) | `index.html` course unlock | Only write on pass |
| `auditLog/{id}` | exam page `_logExamEvent`, `index.html` hub lock | `sf-certs-hub.html` Audit Log tab | Permanent. Admin-only read. |

---

## 12. Debugging Tips

- **Version log:** Add `console.log('[your-exam] v<version> loaded')` at the top of your script
- **slotId check:** Log `slotId` immediately after auth — if null, Write 5 will be silently skipped
- **examId check:** Log `myExamId` after `findMyExamId()` — if null, Write 2 will be skipped and results won't appear in hub reports
- **sessionDocId check:** Log `sessionDocId` after session doc creation — if null, flags won't be saved
- **Firebase SDK version:** Use `12.10.0` (same as hub) to avoid SDK mismatch errors
- **Rules:** Confirm `examSessions`, `examResults`, `examSlots`, `auditLog`, `progress` all allow writes from `isSF()` in Firestore rules v1.8+
