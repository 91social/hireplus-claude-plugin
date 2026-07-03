---
name: interview-oncall
description: Silent interview co-pilot for Hyrewyse. Use when the user says "oncall", "join my interview", "listen to this interview", "I'm interviewing <candidate> now", or starts pasting a live interview transcript. Stays silent while the interview runs, then on "done"/"analyze" checks the candidate's answers against the stored question key points, the resume, and the JD, and delivers a scored assessment.
license: MIT
metadata:
  author: 91social
  version: "1.0.0"
---

# Interview OnCall (silent listener → analyst)

Three phases: **arm → listen (silent) → analyze**. The whole value of this skill is the
discipline of the middle phase: you say nothing useful until the interview is over, so the
interviewer is never distracted and the candidate is never coached.

## Phase 1 — Arm (before or at interview start)

1. **Collect the JD and candidate from the main panel member.** Do NOT guess or silently pick.
   The main panel member (the interviewer who invoked you) must confirm which job and which
   candidate this call is for:
   - If they named both, resolve names → ids (`search_recruitment`, or `list_jobs` →
     `list_resumes`) and **echo back what you resolved** ("Job: Senior Backend Engineer @ Acme,
     Candidate: Jane Doe — correct?").
   - If they named neither or the match is ambiguous, list the options (jobs, then that job's
     candidates) and ask them to pick. Don't go silent until both are confirmed.
2. Pull the evaluation context, in parallel:
   - `get_job` — requirements and responsibilities.
   - `list_resumes` (filtered to the candidate) — resume claims + prior evaluation scores.
   - `get_screening_questions` AND `get_interview_questions` — the planned questions and their
     `key_points` are your scoring rubric.
   - No stored questions? Say so, offer to generate a set first (`generate_interview_questions`,
     or the `generate-interview-questions` skill), or proceed rubric-free scoring against the
     JD alone.
3. Confirm readiness in ONE short line, e.g.:
   `🎧 On call for Jane Doe × Senior Backend Engineer — listening. 8 planned questions loaded. Say "analyze" when done.`

## Phase 2 — Listen (SILENT)

The user pastes transcript chunks, notes, or Q&A as the interview happens.

**Rules — do not break them:**

- Reply to each chunk with a minimal acknowledgment only: `🎧` (nothing else).
- NO commentary, NO partial analysis, NO follow-up suggestions, NO corrections mid-interview
  — even if the candidate says something clearly wrong. Log it mentally for Phase 3.
- Do not call any MCP tools during this phase.
- Only two exits: the user says **"analyze" / "done" / "wrap up"** (→ Phase 3), or asks an
  unrelated question (answer it briefly, then return to listening).
- Exception: if the USER directly asks you for a suggested follow-up question mid-interview
  ("what should I ask next?"), answer with one question, then go silent again.

## Phase 3 — Analyze (on "done" / "analyze")

Work only from the transcript you heard, the stored questions' `key_points`, the resume, and
the JD. Never invent things the candidate didn't say.

Deliver:

1. **Per-question scorecard** — for each planned question that was asked:
   - Verbatim (or near) candidate answer, condensed.
   - Each `key_point`: ✅ covered / 🟡 partially / ❌ missed, with the evidence.
   - A 0–10 answer score.
   Then a list of planned questions that were **never asked**.
2. **Answer verification** — cross-check factual claims made in the interview against the
   resume (`extracted_info`, prior evaluation) and the JD:
   - Consistent claims (interview confirms resume).
   - **Contradictions** (resume says 6 years of X, answer suggests none) — quote both sides.
   - Unverifiable claims worth a reference-check.
3. **Signals** — strengths with evidence, red flags with evidence, and how the interview
   changed (or confirmed) the stored evaluation scores.
4. **Verdict** — advance / hold / reject recommendation with a 2–3 sentence rationale, plus
   3–5 suggested follow-up questions for the next round.
5. **Offer one write-back:** store the follow-up questions for the next round via
   `add_interview_questions` (`question_type: "interview1"`). Ask before writing.

## If MCP tools are missing

The skill still works listen-and-analyze-only (rubric-free, no write-back). For the full flow,
connect the Hyrewyse MCP server (see the `hyrewyse-recruiting` skill) with scopes:
`mcp:jobs.read mcp:resumes.read mcp:questions.read mcp:questions.write`.
