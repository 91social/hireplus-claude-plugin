---
name: generate-screening-questions
description: Generate screening questions for a candidate on Hyrewyse. Use when the user asks to "generate screening questions", "create a phone screen", "screen this candidate", or wants quick go/no-go questions to verify a candidate's basics before a full interview. Generation always uses the platform's AI (generate_screening_questions); Claude authors questions only when the user explicitly asks, and saving over an existing set prompts replace-or-merge.
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
   show it before doing anything else — for the platform path, confirm the user wants a fresh
   set generated; for a set you are about to store yourself, follow
   [Saving when a set already exists](#saving-when-a-set-already-exists).
3. **Pick the generation path.** The platform tool is the ONLY default. Never author questions
   yourself unless the user explicitly asks you to (e.g. "you write them", "don't use the
   platform's generator", "I want Claude-written questions"). Wanting "tailored" or "custom"
   questions is NOT an explicit ask — the platform path takes `extra_topics` for that.

   | Path | When | How |
   |------|------|-----|
   | **Platform (default)** | Every generation request that doesn't explicitly ask otherwise | `generate_screening_questions` with `jd_id`, `resume_id`, optional `num_questions` (default 10) and `extra_topics`. The server generates them with its own AI workflow. |
   | **Claude-authored (explicit request only)** | User explicitly asks YOU to write the questions | Author the questions yourself (playbook below), show the user, then store with `add_interview_questions` (`question_type: "screening"`). |
   | **User-provided** | User supplies their own questions to save | Normalize them into `{ question, key_points }` entries (ask for key points if missing), then store with `add_interview_questions` (`question_type: "screening"`). |

4. **Platform path is async:** it returns `{ question_id, run_id, status: "processing" }`. Poll
   `get_workflow_status` with `run_id` until `COMPLETED`, then fetch the questions with
   `get_screening_questions` and show them.
5. **Show the user** the stored questions with their key points, plus the `question_id`.

## Saving when a set already exists

Applies whenever YOU are about to store questions with `add_interview_questions` (user-provided
or explicitly Claude-authored) and `get_screening_questions` shows an existing set. Never decide
silently — show the latest existing set and ask the user:

- **Replace** — store only the new questions as a new set. It becomes the newest set, which is
  what readers (the list commands, `interview-oncall`) use. The API has no delete, so older sets
  remain in history; mention this.
- **Merge** — combine the latest existing set's questions with the new ones (drop exact
  duplicates), and store the combined list as one new set.

If no set exists yet, just save — no prompt needed.

## Claude-authored playbook

Use ONLY when the user has explicitly asked you to author the questions yourself.

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
