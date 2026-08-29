# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This repository currently contains only the PRD (`1인1역_배정앱_PRD.md`) — no code has been written yet. Treat the PRD as the spec of record until implementation begins; the summary below distills the architectural decisions from it so you don't have to re-read the whole document for routine work. Re-check the PRD directly for anything not covered here, and update this file once real code/config exist (build scripts, actual file layout, etc.) since none of the "commonly used commands" apply yet.

## What this app is

A classroom role-assignment tool ("1인1역 배정") for Korean elementary/middle school homeroom teachers. Teachers collect students' top 1–3 role preferences via a QR code, then the app computes an assignment that maximizes class-wide satisfaction (not just pairwise conflict resolution). Teachers review, manually adjust, and finalize the result themselves — the app never auto-decides.

## Mandated tech stack (do not deviate without asking)

- **Single `index.html` file, no build tooling.** No bundler, no package.json, no framework toolchain.
- **CDN-only dependencies**, loaded directly in `index.html`.
- A top-level `CONFIG` object at the very top of the file for tunables (e.g. cost weights, Firebase config).
- **Korean inline comments** throughout.
- Deploy target: **GitHub Pages**.
- Auth/data: **Firebase Auth** (Google sign-in, teachers only) + **Firestore**.

Because there's no build step, "development" means editing `index.html` directly and testing by opening it (or serving it statically) in a browser.

## Firebase project reuse — important constraint

**Do not create a new Firebase project.** This app is added as a new set of collections (prefixed `roleAssign_`) inside the existing "조회 도우미 (Johoe)" Firebase project, to stay within the free-tier project-count limit. Firestore collections are fully isolated from each other, so this is safe — just make sure:
- All collection names for this app are prefixed `roleAssign_` to avoid colliding with the existing app's data.
- Security rules for this app's collections are scoped narrowly and don't broaden access to the existing app's data.

## Data model

```
roleAssign_classes/{classId}
  ├─ ownerUid, className, submissionOpen, createdAt/updatedAt
  ├─ students/{studentId}       — number (학번), name
  ├─ roles/{roleId}             — name, capacity
  ├─ preferences/{studentNumber} — choices: [roleId, roleId, roleId], submittedAt
  └─ drafts/{draftId}           — assignments: {studentNumber: roleId|null},
                                   stats: {first, second, third, unassigned},
                                   isFinal, createdAt
```

Multi-tenant: each class's data (roster, roles, preferences, drafts) lives entirely under its `classId` and is scoped to `ownerUid`. Teachers never see each other's classes.

## Write policy

- **Field-level merge writes only — never overwrite whole documents.** Multiple students submit preferences concurrently; a full-document overwrite would cause last-write-wins data loss.
- Each student's submission is its own independent document: `preferences/{학번}`.

## Security rules (intent, not yet implemented)

- `roleAssign_classes/{classId}`: read/write only if `ownerUid == request.auth.uid`.
- `preferences/{studentNumber}` subcollection: unauthenticated writes allowed, but **only to the student's own 학번 document**, and only while `submissionOpen == true`.
- Students can never read other students' preferences or any draft/assignment results.
- A submission for a 학번 not present in the class roster must be rejected.

## Assignment algorithm

Goal: maximize the number of students getting their 1st-choice role across the whole class (not just resolve conflicts pairwise), with ties broken randomly.

- Model as **min-cost max-flow** (or capacity-expanded Hungarian algorithm).
- Costs: 1st choice = 0, 2nd choice = 10, 3rd choice = 100. The wide gap between weights is intentional — it prevents sacrificing one 1st-choice placement to gain several 2nd-choice placements; higher-choice counts are maximized first, lexicographically.
- Tie-breaking: shuffle student input order randomly on each run so that when multiple optimal solutions exist (equal total cost), one is chosen at random.
- Runs entirely **client-side**; must complete instantly for ~30 students.
- Students who submitted but couldn't get any of their 3 choices, and students who never submitted, are left **unassigned** — never auto-filled into leftover roles. They're surfaced as a separate "미배정" list for the teacher to assign by hand.
- Each run produces a new saved draft (안 1, 안 2, …) with summary stats (1st/2nd/3rd/unassigned counts) so teachers can compare multiple runs before picking one to finalize and manually adjust.

## Key product rules to preserve

- **Results are teacher-only.** Students never see assignment outcomes in the app (deliberately, to avoid classroom disruption from staggered reveals) — the teacher announces results in person. Don't add a student-facing results view without being asked.
- Role capacity total may be ≥ student count (leftover seats stay empty) but the UI must warn when capacity total < student count, and the warning should link directly to the capacity-editing screen.
- Once a draft is finalized (`isFinal`), it's locked; unlocking it for re-edit is an explicit separate action.
- Student login is 학번-only (no password) but the roster must store both 학번 and name, because the student flow shows the matched name so students can self-verify they entered the right 학번.
