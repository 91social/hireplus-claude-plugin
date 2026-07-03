---
name: generate-interview-questions
description: Generate round-1 interview questions for a candidate on Hyrewyse. Use when the user asks to "generate interview questions", "prepare the technical interview", "what should I ask this candidate in the interview", or wants deep technical/behavioral questions tailored to a specific candidate and job. Supports the platform's own AI generation (generate_interview_questions) or Claude-authored questions.
license: MIT
metadata:
  author: 91social
  version: "1.1.0"
---

# Generate Interview Questions

Interview questions (`question_type: "interview1"`) are the **deep evaluation round**: technical
depth, problem-solving, and behavioral evidence, asked by an interviewer who knows the domain.
For the quick recruiter phone-screen instead, use the `generate-screening-questions` skill.

## Flow

1. **Resolve the candidate and job.** You need a `jd_id` and `resume_id`.
   - Names → ids: `search_recruitment`, or `list_jobs` → `list_resumes` (filter by
     `candidate_name`). Ambiguous matches → show them and ask.
2. **Check prior rounds:**
   - `get_screening_questions` — don't re-ask what the screen already covered; build on it.
   - `get_interview_questions` — if a set already exists, show it and confirm before writing
     another.
3. **Pick the generation path:**

   | Path | When | How |
   |------|------|-----|
   | **Platform (default)** | User just wants interview questions | `generate_interview_questions` with `jd_id`, `resume_id`, optional `num_questions` (default 10). The server generates technical Level-1 questions with its own AI workflow. |
   | **Claude-authored** | User wants tailored/custom questions, or wants to review before storing | Author the questions yourself (playbook below), show the user, then store with `add_interview_questions` (`question_type: "interview1"`). |

4. **Platform path is async:** it returns `{ question_id, run_id, status: "processing" }`. Poll
   `get_workflow_status` with `run_id` until `COMPLETED`, then fetch the questions with
   `get_interview_questions` and show them.
5. **Show the user** the stored set, with key points and the `question_id`.

## Claude-authored playbook

Generate 8–10 questions anchored in THIS candidate's background — never generic. Use `get_job`
(requirements, responsibilities) and the candidate's `list_resumes` row (evaluation scores,
`skills_matched`, `key_skills_missing`) as inputs. Mix:

- **Depth probes on matched skills** (3–4): take the candidate's strongest claimed
  skills/projects and go deep — architecture choices, trade-offs, failure modes, "what would
  you change now". A strong candidate has real answers; an inflated resume falls apart here.
- **Gap probes** (1–2): for each key missing skill, test adjacency — can they reason about it
  from first principles even without direct experience?
- **Role-responsibility scenarios** (2–3): turn the JD's `responsibilities` into concrete
  "how would you…" scenarios at this company's scale.
- **Behavioral** (1–2): conflict, ownership, mentorship — matched to the seniority the JD asks for.
- Every question gets 2–4 `key_points`: what a strong answer contains — specific techniques,
  trade-offs named, honest limits acknowledged.

Quality bar:

- Each question must reference something real — a project on the resume, a JD responsibility, a
  score gap in the evaluation. If a question would work for any candidate, rewrite it.
- Order the list as an interview arc: warm-up depth probe first, hardest scenario mid-way,
  behavioral near the end.

## If MCP tools are missing

Tell the user to connect the Hyrewyse MCP server (see the `hyrewyse-recruiting` skill). Required
scopes: `mcp:jobs.read mcp:resumes.read mcp:questions.read mcp:questions.write mcp:workflows.read`.
