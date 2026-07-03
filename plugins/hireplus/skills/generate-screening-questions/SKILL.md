---
name: generate-screening-questions
description: Generate screening questions for a candidate on Hyrewyse. Use when the user asks to "generate screening questions", "create a phone screen", "screen this candidate", or wants quick go/no-go questions to verify a candidate's basics before a full interview. Supports the platform's own AI generation (generate_screening_questions) or Claude-authored questions.
license: MIT
metadata:
  author: 91social
  version: "1.0.0"
---

# Generate Screening Questions

Screening questions are the **first-pass filter**: short, factual, verifiable questions a recruiter
(often non-technical) asks in a 15–30 minute call to decide whether the candidate advances to a real
interview. They are NOT deep technical questions — that's the `generate-interview-questions` skill.

## Flow

1. **Resolve the candidate and job.** You need a `jd_id` and `resume_id`.
   - Names → ids: `search_recruitment`, or `list_jobs` → `list_resumes` (filter by
     `candidate_name`). Ambiguous matches → show them and ask.
2. **Check for existing questions** with `get_screening_questions`. If a set already exists,
   show it and ask whether to generate a fresh set before writing another.
3. **Pick the generation path:**

   | Path | When | How |
   |------|------|-----|
   | **Platform (default)** | User just wants screening questions | `generate_screening_questions` with `jd_id`, `resume_id`, optional `num_questions` (default 10) and `extra_topics`. The server generates them with its own AI workflow. |
   | **Claude-authored** | User wants tailored/custom questions, or wants to review before storing | Author the questions yourself (playbook below), show the user, then store with `add_interview_questions` (`question_type: "screening"`). |

4. **Platform path is async:** it returns `{ question_id, run_id, status: "processing" }`. Poll
   `get_workflow_status` with `run_id` until `COMPLETED`, then fetch the questions with
   `get_screening_questions` and show them.
5. **Show the user** the stored questions with their key points, plus the `question_id`.

## Claude-authored playbook

Generate 6–8 screening questions. Target mix:

- **Experience verification** (2–3): confirm the resume's claims — years with the core required
  skills, role scope, team size. Ground each question in a specific resume claim (use `get_job` +
  the candidate's `list_resumes` row for context).
- **Skill-gap probes** (1–2): one question per item in the evaluation's `key_skills_missing` —
  has the candidate touched it at all, or is the gap real?
- **Logistics & fit** (2–3): notice period / availability, location vs the job's
  `location`/`remote_policy`, salary expectations vs `salary_range` (if the JD has one).
- Every question gets 2–4 `key_points`: the concrete facts a screener should hear in a good
  answer (numbers, names, dates — not vibes).

Quality bar:

- A screener must be able to judge each answer **without domain expertise** — key points are
  checkable facts ("names a specific Postgres version or feature", "notice period ≤ 30 days").
- Never ask what the resume already answers unambiguously; ask what needs **verifying**.
- Keep each question one sentence. No multi-part questions in a screen.

## If MCP tools are missing

Tell the user to connect the Hyrewyse MCP server (see the `hyrewyse-recruiting` skill). Required
scopes: `mcp:jobs.read mcp:resumes.read mcp:questions.read mcp:questions.write mcp:workflows.read`.
