---
description: List the stored screening questions for a candidate on Hyrewyse
argument-hint: <candidate name or resume_id> [job title or jd_id]
---

List the stored SCREENING questions for the candidate given in: $ARGUMENTS

1. Resolve `jd_id` and `resume_id`:
   - If the arguments already look like ids (UUIDs), use them directly.
   - Otherwise resolve names: `search_recruitment` with the candidate name, or `list_jobs`
     (filter by job title) → `list_resumes` (filter by `candidate_name`).
   - If no arguments were given, ask which candidate and job.
   - If several candidates/jobs match, show the matches and ask the user to pick one.
2. Call `get_screening_questions` with the resolved `jd_id` and `resume_id`.
3. Present the result:
   - No sets → say none exist and offer to generate one with the platform's
     `generate_screening_questions` (see the `generate-screening-questions` skill).
   - A set with `questions_generated: null` → generation is still running; offer to poll
     `get_workflow_status`.
   - Otherwise render each set (newest first) as a numbered list: the question, then its
     `key_points` as sub-bullets. Include the set's `question_id` and creation time.
