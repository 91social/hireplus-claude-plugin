---
name: hyrewyse-recruiting
description: Drive the Hyrewyse recruitment platform through its MCP server. Use when the user wants to create a job description, add or screen candidate resumes, score a candidate against a job, generate interview questions, or browse jobs/resumes on Hyrewyse. Critically, this skill teaches the "client-side intelligence" write tools (create_job, create_resume, add_interview_questions) where YOU extract the structured data and scores from the source text and submit them — the server runs no LLM of its own for these.
license: MIT
metadata:
  author: 91social
  version: "1.0.0"
  repository: https://github.com/91social/hyrewyseplatform
---

# Hyrewyse Recruiting (MCP)

Hyrewyse is an AI recruitment platform. Its backend exposes a remote **MCP server** with OAuth 2.1
auth. This skill teaches you to use those tools well — especially the **client-side intelligence**
write tools, where *you* (the calling agent) do the extraction and scoring and submit structured
data for direct storage.

## When to use this skill

Use it when the user asks to:

- Create / register a **job description** (JD) on Hyrewyse.
- Add a **candidate resume** to a job, with or without an evaluation score.
- **Score / evaluate** a candidate against a job.
- Generate **interview questions** for a candidate.
- Browse jobs or candidates (`list_jobs`, `get_job`, `list_resumes`).
- Check the status of an async upload (`get_workflow_status`).

If the Hyrewyse MCP tools are not connected, tell the user to connect the Hyrewyse MCP server first
(see [Connecting](#connecting-scopes--auth)).

## The core idea: client-side intelligence

Hyrewyse has **two write paths**. Pick deliberately:

| You have… | Use | Who extracts / scores |
|-----------|-----|-----------------------|
| The JD / resume **text** (or you can read it) | `create_job`, `create_resume`, `add_interview_questions` | **YOU** — read the text, produce the structured fields + scores, submit them. Stored immediately. |
| A raw **file** (PDF/DOCX bytes) you can't read here | `upload_resume` | **The server** — runs async Mastra extraction + evaluation. You then poll `get_workflow_status`. |

For the client-side tools, the server **trusts and stores** the structured payload you send. There is
no server-side LLM cleanup. So the quality of the JD fields, the parsed resume, and the candidate
score is entirely **your** responsibility. Extract carefully and score honestly.

> Rule of thumb: if you can see/read the source text, prefer `create_resume` (you extract + score) so
> the record is complete instantly. Only fall back to `upload_resume` when all you have is an opaque
> file you cannot parse in this session.

## Tool catalog

Reads:

| Tool | Scope | Purpose |
|------|-------|---------|
| `list_jobs` | `mcp:jobs.read` | List JDs, with filters (`company`, `job_title`, `location`, `status`) + pagination. |
| `get_job` | `mcp:jobs.read` | Fetch one JD (with its `extracted_info`) by `jd_id`. |
| `list_resumes` | `mcp:resumes.read` | List candidates for a JD (with scores), filter by `candidate_name`, `email`, `hiring_status`. |
| `get_screening_questions` | `mcp:questions.read` | Fetch stored screening questions (with `key_points`) for a candidate. |
| `get_interview_questions` | `mcp:questions.read` | Fetch stored interview (level 1) questions (with `key_points`) for a candidate. |
| `get_workflow_status` | `mcp:workflows.read` | Status of an async run (`run_id`) started by `upload_resume` or the `generate_*_questions` tools. |

Writes — client-side intelligence (you supply structured data):

| Tool | Scope | You supply |
|------|-------|------------|
| `create_job` | `mcp:jobs.write` | `raw_text` + `extracted_info` (parsed JD fields). |
| `create_resume` | `mcp:resumes.write` | `jd_id` + `raw_text` + `extracted_info` (parsed resume) + `evaluation` (your scoring). |
| `add_interview_questions` | `mcp:questions.write` | `jd_id` + `resume_id` + `questions[]` (each with `key_points`). |

Writes — server-side intelligence (the platform's own AI does the work, async):

| Tool | Scope | You supply |
|------|-------|------------|
| `upload_resume` | `mcp:resumes.write` | `jd_id` + `filename` + `content_base64`. Returns `status: "processing"`. |
| `generate_screening_questions` | `mcp:questions.write` | `jd_id` + `resume_id` (+ `num_questions`, `extra_topics`). Returns `run_id` to poll. |
| `generate_interview_questions` | `mcp:questions.write` | `jd_id` + `resume_id` (+ `num_questions`). Returns `run_id` to poll. |

Full field-by-field schemas and copy-paste JSON examples are in
[references/tool-reference.md](references/tool-reference.md). Read it before your first write call.

## The recruiting flow

```
create_job ──► (per candidate) create_resume ──► add_interview_questions
   │                  │  extract + score the candidate          │  generate Q's from JD + resume
   └─ jd_id           └─ resume_id                              └─ stored against (jd_id, resume_id)
```

1. **Create the job.** Read the JD text. Extract `job_title`, `company`, `required_skills`,
   `responsibilities`, etc. Call `create_job`. Keep the returned `jd_id`.
2. **Add each candidate.** Read the resume. Extract the candidate fields. Then **evaluate the
   candidate against the JD** — produce the 0–100 scores, `skills_matched` / `key_skills_missing`,
   a `final_recommendation`, and a short `evaluation_summary`. Call `create_resume`. Keep `resume_id`.
3. **Generate questions.** Two ways: let the platform generate them (`generate_screening_questions`
   / `generate_interview_questions`, async — poll `get_workflow_status`), or author them yourself
   and store with `add_interview_questions` (`question_type` `"screening"` or `"interview1"`).
   Before writing, check `get_screening_questions` / `get_interview_questions` so you don't
   duplicate an existing set. The dedicated `generate-screening-questions` and
   `generate-interview-questions` skills carry the full playbooks; `interview-oncall` uses the
   stored questions as a live-interview rubric.

## Doing the extraction & scoring well

- **JD `extracted_info`:** `job_title` and `company` are required. Fill `required_skills`,
  `responsibilities`, `qualifications` from the text — don't invent. Use `null` for unknown
  single-value fields (`salary_range`, `education_required`), `[]` for unknown lists.
- **Resume `extracted_info`:** `candidate_name` is required. Capture `skills`, `total_experience`
  (years, decimal), and the `experience` / `education` arrays. Leave unknowns `null` / `[]`.
- **`evaluation` (scoring):** Be objective and base scores on concrete evidence in the resume vs the
  JD's `required_skills`.
  - `overall_score` — holistic fit (0–100).
  - `role_fit_score` — experience vs the role's responsibilities.
  - `technical_skill_score` — skills vs the JD's required skills.
  - `skills_matched` / `key_skills_missing` — drawn from the JD's required skills.
  - `final_recommendation` — exactly one of `Strong Yes`, `Yes`, `Maybe`, `No`, `Strong No`.
  - `evaluation_summary` — 2–3 sentences on strengths and gaps.

## Reading results back

- A candidate created via `create_resume` is stored `completed` and appears in `list_resumes` for its
  JD **with the evaluation joined** (`overall_score`, `final_recommendation`, etc.).
- `create_resume` **dedupes** on the resume text per job: a re-submit of the same `raw_text` returns
  `{ status: "skipped", resume_id: <existing> }`. Treat that as success, not an error.
- A missing `jd_id` (or `resume_id` for questions) returns `{ status: "error", message }` — surface the
  message; don't retry blindly.

## upload_resume (raw file) path

Only when you cannot read the file yourself:

1. `upload_resume` with base64 bytes → returns `{ resume_id, status: "processing" }`.
2. The server extracts + scores asynchronously. The resume is **not** in `list_resumes` until it
   finishes (that list shows only `completed`).
3. Poll `get_workflow_status` with the run id to know when it's done.

## Connecting (scopes & auth)

The Hyrewyse MCP server is an OAuth 2.1 resource server. Connect it in your MCP client; sign in with
your Hyrewyse email + password on the consent page. Request only the scopes you need:

- Browsing only → `mcp:jobs.read mcp:resumes.read`
- Full recruiting flow → `mcp:jobs.read mcp:jobs.write mcp:resumes.read mcp:resumes.write mcp:questions.read mcp:questions.write mcp:workflows.read`
- (`mcp:workflows.read` is needed to poll `upload_resume` and `generate_*_questions` runs)

A `403 insufficient_scope` means your token lacks the scope for that tool — re-authorize with the
scope named in the `WWW-Authenticate` header. A `401` means the access token is missing/expired —
re-run the OAuth flow.
