# Hyrewyse MCP — Tool Reference

Exhaustive input/output schemas for every Hyrewyse MCP tool, with copy-paste JSON arguments.
Field types use TypeScript notation. "required" means the call is rejected without it.

---

## create_job  (scope: `mcp:jobs.write`)

Create a job description from fields **you** extracted from the JD text. Stored as `completed`.

**Arguments**

| Field | Type | Req | Notes |
|-------|------|-----|-------|
| `raw_text` | string | ✅ | The original JD text you extracted from. |
| `extracted_info` | object | ✅ | Parsed JD, schema below. |
| `added_by` | string | — | Defaults to `"mcp"`. |

`extracted_info`:

| Field | Type | Req |
|-------|------|-----|
| `job_title` | string | ✅ |
| `company` | string | ✅ |
| `location` | string | — |
| `employment_type` | string | — | e.g. Full-time, Contract |
| `experience_required` | string | — | e.g. "3-5 years" |
| `salary_range` | string \| null | — |
| `required_skills` | string[] | — |
| `nice_to_have_skills` | string[] | — |
| `responsibilities` | string[] | — |
| `qualifications` | string[] | — |
| `education_required` | string \| null | — |
| `department` | string \| null | — |
| `remote_policy` | string | — | Remote / Hybrid / On-site |

**Example**

```json
{
  "raw_text": "Senior Backend Engineer at Acme Corp (Bangalore, Hybrid). 5+ years building APIs. Must know TypeScript, Node.js, Postgres. Nice: Kafka, AWS. You'll own services end-to-end and mentor juniors. Bachelor's in CS or equivalent.",
  "extracted_info": {
    "job_title": "Senior Backend Engineer",
    "company": "Acme Corp",
    "location": "Bangalore, India",
    "employment_type": "Full-time",
    "experience_required": "5+ years",
    "salary_range": null,
    "required_skills": ["TypeScript", "Node.js", "Postgres"],
    "nice_to_have_skills": ["Kafka", "AWS"],
    "responsibilities": ["Own backend services end-to-end", "Mentor junior engineers"],
    "qualifications": ["5+ years backend development"],
    "education_required": "Bachelor's in Computer Science or equivalent",
    "department": "Engineering",
    "remote_policy": "Hybrid"
  },
  "added_by": "recruiter@acme.com"
}
```

**Returns:** `{ "jd_id": "<uuid>", "status": "completed" }` — keep `jd_id`.

---

## create_resume  (scope: `mcp:resumes.write`)

Add a candidate to a job using a resume **you** parsed plus an evaluation **you** produced against
the JD. Stores the resume (`completed`) and a candidate evaluation in one call.

**Arguments**

| Field | Type | Req | Notes |
|-------|------|-----|-------|
| `jd_id` | string | ✅ | Must be an existing job. |
| `raw_text` | string | ✅ | Original resume text. Also used for dedupe. |
| `extracted_info` | object | ✅ | Parsed resume, schema below. |
| `evaluation` | object | ✅ | Your scoring, schema below. |
| `added_by` | string | — | Defaults to `"mcp"`. |

`extracted_info`:

| Field | Type | Req |
|-------|------|-----|
| `candidate_name` | string | ✅ |
| `email` | string \| null | — |
| `phone` | string \| null | — |
| `location` | string \| null | — |
| `summary` | string \| null | — |
| `total_experience` | number | — | Years, decimal (e.g. 5.5) |
| `current_title` | string \| null | — |
| `current_company` | string \| null | — |
| `skills` | string[] | — |
| `companies_worked_for` | string[] | — |
| `experience` | `{ company, title, start_date, end_date, description }[]` | — | All sub-fields optional strings |
| `education` | `{ institution, degree, field, graduation_year }[]` | — | All sub-fields optional strings |
| `certifications` | string[] | — |
| `languages` | string[] | — |

`evaluation` (all required):

| Field | Type | Notes |
|-------|------|-------|
| `overall_score` | number 0–100 | Holistic fit |
| `role_fit_score` | number 0–100 | Experience vs role |
| `technical_skill_score` | number 0–100 | Skills vs required skills |
| `key_skills_missing` | string[] | Required skills the candidate lacks |
| `skills_matched` | string[] | Required skills the candidate has |
| `final_recommendation` | `"Strong Yes" \| "Yes" \| "Maybe" \| "No" \| "Strong No"` | Exactly one |
| `evaluation_summary` | string | 2–3 sentences |

**Example**

```json
{
  "jd_id": "3f2a…",
  "raw_text": "Jane Doe — jane@example.com. 6 years backend. Staff Engineer at Globex. Built high-throughput TypeScript/Node services on Postgres. Led a team of 5. B.Tech CSE, 2017.",
  "extracted_info": {
    "candidate_name": "Jane Doe",
    "email": "jane@example.com",
    "phone": null,
    "location": null,
    "summary": "Backend engineer with 6 years building high-throughput services.",
    "total_experience": 6,
    "current_title": "Staff Engineer",
    "current_company": "Globex",
    "skills": ["TypeScript", "Node.js", "Postgres", "Team leadership"],
    "companies_worked_for": ["Globex"],
    "experience": [
      { "company": "Globex", "title": "Staff Engineer", "start_date": "2020", "end_date": "present", "description": "Built high-throughput services; led a team of 5." }
    ],
    "education": [
      { "institution": "IIT", "degree": "B.Tech", "field": "Computer Science", "graduation_year": "2017" }
    ],
    "certifications": [],
    "languages": ["English"]
  },
  "evaluation": {
    "overall_score": 88,
    "role_fit_score": 90,
    "technical_skill_score": 85,
    "key_skills_missing": [],
    "skills_matched": ["TypeScript", "Node.js", "Postgres"],
    "final_recommendation": "Strong Yes",
    "evaluation_summary": "Strong fit: 6 years of backend work on exactly the required stack, plus team leadership. No required skills missing; light on the nice-to-have Kafka/AWS."
  },
  "added_by": "recruiter@acme.com"
}
```

**Returns:**
- Success: `{ "resume_id": "<uuid>", "status": "completed", "overall_score": 88 }`
- Duplicate (same `raw_text` for this job): `{ "resume_id": "<existing>", "status": "skipped", "message": "Duplicate resume" }` — treat as success.
- Bad job: `{ "resume_id": null, "status": "error", "message": "Job <id> not found" }`.

---

## add_interview_questions  (scope: `mcp:questions.write`)

Store interview questions **you** generated for a candidate against a job. The candidate must already
exist under the job (create it first with `create_resume` or `upload_resume`).

**Arguments**

| Field | Type | Req | Notes |
|-------|------|-----|-------|
| `jd_id` | string | ✅ | Existing job. |
| `resume_id` | string | ✅ | Existing candidate under that job. |
| `question_type` | `"screening" \| "interview1"` | — | Defaults to `"screening"`. |
| `questions` | `{ question, key_points }[]` | ✅ | At least 1. `key_points`: 2–4 things to listen for. |

**Example**

```json
{
  "jd_id": "3f2a…",
  "resume_id": "9b71…",
  "question_type": "screening",
  "questions": [
    {
      "question": "Walk me through a high-throughput service you built on Postgres. How did you handle connection pooling and slow queries?",
      "key_points": ["Concrete throughput numbers", "Pooling strategy", "Query profiling / indexing", "Trade-offs made"]
    },
    {
      "question": "You led a team of 5 at Globex. Describe a technical disagreement and how you resolved it.",
      "key_points": ["Listens to others", "Data-driven decision", "Ownership of outcome"]
    }
  ]
}
```

**Returns:** `{ "question_id": "<uuid>", "status": "completed" }`. Missing job/resume →
`{ "question_id": null, "status": "error", "message": "…" }`.

---

## upload_resume  (scope: `mcp:resumes.write`)

Raw-file path. Use **only** when you cannot read the file yourself. The server extracts and scores
asynchronously.

**Arguments**

| Field | Type | Req | Notes |
|-------|------|-----|-------|
| `jd_id` | string | ✅ | Existing job. |
| `filename` | string | ✅ | e.g. `cv.pdf` — drives the parser. |
| `content_base64` | string | ✅ | Base64 of the file bytes. |
| `added_by` | string | — | Defaults to `"mcp"`. |

**Returns:** `{ "resume_id": "<uuid>", "status": "processing" }`, or
`{ "resume_id": "<existing>", "status": "skipped", "message": "Duplicate resume" }`. Poll
`get_workflow_status` to know when extraction/evaluation finishes; the resume only shows in
`list_resumes` once `completed`.

---

## list_jobs  (scope: `mcp:jobs.read`)

| Field | Type | Notes |
|-------|------|-------|
| `page` | number | ≥1 |
| `per_page` | number | 1–100 (default 20) |
| `company` | string | Substring match |
| `job_title` | string | Substring match |
| `location` | string | Substring match |
| `status` | string | `uploaded` \| `processing` \| `completed` \| `failed` |

Returns a paginated `{ items, total, page, per_page, total_pages, has_next, has_prev }`.

## get_job  (scope: `mcp:jobs.read`)

`{ "jd_id": "<id>" }` → the JD row including `extracted_info`, or an error if not found.

## list_resumes  (scope: `mcp:resumes.read`)

| Field | Type | Notes |
|-------|------|-------|
| `jd_id` | string | Required |
| `page` / `per_page` | number | per_page 1–100 |
| `candidate_name` | string | Substring match |
| `email` | string | Substring match |
| `hiring_status` | string | `Evaluating` \| `Screening` \| `ShortListed` \| `Hired` \| `Rejected` \| `Interview 1` \| `Interview 2` |

Returns candidates **with evaluation fields merged** (`overall_score`, `role_fit_score`,
`technical_skill_score`, `skills_matched`, `key_skills_missing`, `final_recommendation`). Only
`completed` resumes are listed.

## get_workflow_status  (scope: `mcp:workflows.read`)

`{ "run_id": "<id>" }` → `{ id, status: "PENDING" | "RUNNING" | "COMPLETED" | "FAILED", … }`.

---

## Scope → tool map (for OAuth)

| Scope | Grants |
|-------|--------|
| `mcp:jobs.read` | `list_jobs`, `get_job` |
| `mcp:jobs.write` | `create_job` |
| `mcp:resumes.read` | `list_resumes` |
| `mcp:resumes.write` | `create_resume`, `upload_resume` |
| `mcp:questions.write` | `add_interview_questions` |
| `mcp:workflows.read` | `get_workflow_status` |
