# ADR-017: Question pool generation, lifecycle, and scoring

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 5 (AI Assessment); planner pre-work in Phase 3

## Context

The differentiating capability of the platform (CLAUDE.md §1) is
**per-student AI assessment**. The instructor wants:

- Consistent quality across attempts (same task → comparable rigor).
- The ability to review and edit what the AI is going to ask before
  any student sees it.
- Defensible, transparent scoring — they can show a student exactly
  why they got the score they got.
- An override path when the AI is wrong.
- A predictable cost profile — we cannot generate a fresh pool per
  attempt at scale.

## Decision

### Pool lifecycle

```
Task (PUBLISHED)
   │
   ▼
PlannerAgent.proposeQuestionPool(taskId, n, difficultyMix)
   │
   ▼
QuestionPool (status=DRAFT) + N Questions
   │
   ▼  instructor reviews / edits / regenerates per question;
   │  sets per-question rubric_json
   ▼
QuestionPool (status=REVIEWED)
   │
   ▼  instructor publishes
   ▼
QuestionPool (status=PUBLISHED) ← attempts can now sample from this
```

- Pool generation is a **task-level event**, not per-attempt.
- Regeneration is per-question (preferred) or whole-pool (allowed).
  Both write a new `Question` row and flip `instructor_edited` on
  edit. The previous version is preserved for audit.
- A task may have multiple historical pools but only one with status
  `PUBLISHED` at a time.

### Per-attempt sampling

- On attempt start, the system selects a **random subset** of size
  `attempt_size` (configurable per task; default 5 of 20).
- Sampling respects a difficulty mix the instructor configures on
  the task (e.g., 2 EASY / 2 MED / 1 HARD).
- The same student does not see identical samples on consecutive
  attempts (best-effort dedupe within a sliding window).
- MCQ and TEXT questions are interleaved per the configured mix.

### MCQ scoring

- UI renders clickable options only; the text input is disabled.
  The AssessorAgent has no tool to accept free text for an MCQ.
- Scored by exact match on `correct_idx`.
- AssessorAgent invariant (enforced in code): cannot reveal the
  correct option in a follow-up. The UI hides correctness until the
  attempt is finalized.

### TEXT scoring

- Free-text input. AssessorAgent may call `askFollowUp(reason)` up
  to **2 times per question** if the rubric flags the answer as
  ambiguous (e.g., "did the student mean X or Y?"). The cap is
  enforced in the agent loop, not in the prompt.
- Once the agent commits to a score, it calls `scoreAnswer(questionId,
  answer, rubric)`. The rubric is an array of criteria:

  ```jsonc
  [
    { "id": "correctness", "max": 5, "description": "..." },
    { "id": "depth",       "max": 3, "description": "..." },
    { "id": "clarity",     "max": 2, "description": "..." }
  ]
  ```

- `scoreAnswer` returns per-criterion `{ id, awarded, justification }`.
- Question score = sum of awarded across criteria. Persisted on the
  attempt-question record.

### Final score

- `final_score` = weighted sum of question scores (weights default
  equal; instructor can configure per-question weight).
- Pass/fail is computed from a task-level threshold; not stored
  separately, derived from `final_score` for display.

### Instructor override

- Instructor opens any completed attempt and sees the full transcript
  (all `AttemptTurn` rows + per-criterion justifications + AI score).
- They can write a single `instructor_override_score` (with optional
  remark). Both AI and override are kept; UI shows override as the
  effective score with a badge.
- Override action writes to `AuditLog`
  (`assessment.override`).

### AssessorAgent invariants (enforced in code, not prompt)

- Cannot reveal MCQ correct answers mid-attempt.
- Cannot mutate `Question.payload_json`.
- Cannot finalize an attempt without going through `scoreAnswer` for
  every question presented.
- Tool calls fail closed: any Zod-invalid arg → tool errors → agent
  retries within turn budget → attempt errors and is paused for
  instructor inspection if budget is exhausted.

## Consequences

### Positive

- **Cost predictable**: pool generation is once per task, not per
  attempt. A class of 100 students sharing a pool of 20 costs ~the
  same as a class of 10.
- **Quality controllable**: instructor reviews questions before any
  student sees them; bad outputs get fixed once, not 100 times.
- **Auditable**: every attempt has a per-criterion breakdown +
  transcript + Portkey trace; disputes are resolvable.
- **Fair within a cohort**: random sampling from a fixed pool means
  variance comes from sampling, not from prompt drift.
- **Override is first-class**: AI gets it wrong sometimes; we don't
  pretend otherwise.

### Negative / Trade-offs

- Random sampling can produce unlucky distributions (a student gets
  three HARD in a row even with a configured mix). Difficulty-quota
  sampling mitigates but doesn't eliminate.
- Two-pass review (generate → review → publish) adds an instructor
  step before students can attempt a task. This is intentional but
  needs clear UI to avoid being annoying.
- Rubric authoring is real work. We provide a default rubric template
  and let the planner suggest one as a starting point.

### Neutral

- Pool size, attempt size, and difficulty mix are task-level config.
- Regeneration of a single question keeps the prior version row
  (soft-replaced) so history is preserved.
- Follow-up cap is a constant (`MAX_FOLLOWUPS_PER_QUESTION = 2`)
  defined in `packages/llm/agents/assessor/constants.ts`.

## Alternatives considered

- **Per-attempt question generation**: maximally fresh, but
  unpredictable cost and impossible for the instructor to QA. Hidden
  cost of debugging "why did the AI ask this" multiplied by every
  attempt.
- **Single 0–100 holistic score** instead of rubric breakdown:
  cheaper to render, but indefensible when a student asks "why".
- **Pass/fail only**: loses the signal we need for cohort
  differentiation and per-domain feedback.
- **Pre-canned question banks (no LLM generation)**: requires manual
  authoring at scale; loses the "personalized to learning objectives"
  benefit that's the entire point.

## Related ADRs / docs

- `CLAUDE.md` §6 (LLM strategy — Agents, Token tracking).
- ADR-005 (LLM gateway).
- ADR-011 (Realtime transport) — assessor message stream is SSE.
- ADR-012 (Observability) — `portkey_trace_id` per turn.
- ADR-016 (Data model) — `Task`, `QuestionPool`, `Question`,
  `AssessmentAttempt`, `AttemptTurn`.
- `docs/development/phase_3.md` (planner tools including
  `proposeQuestionPool`) and `docs/development/phase_5.md` (assessor
  + scoring).
