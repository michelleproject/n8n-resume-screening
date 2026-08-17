# Resume Screening Workflow

An n8n workflow that reads resumes from Gmail, scores each candidate against role
criteria using Gemini, routes them into shortlist / maybe / reject, and logs every
result to Google Sheets.

Built to solve a specific problem: resumes get read in the order they arrived, 
and reviewer attention drops off after the first twenty or thirty resumes in a 
sitting. This reads every one with the same depth.

## What it does

1. Pulls up to 10 Gmail messages matching `has:attachment filename:pdf -label:screened`
2. Downloads the PDF attachment and extracts its text
3. Injects the role criteria (title, must-have skills, minimum years, nice to have)
4. Sends resume plus criteria to Gemini, which returns structured JSON
5. Routes on the verdict field into one of three branches
6. Appends a row to Google Sheets for every candidate, including rejects
7. Labels the email so it never gets screened twice

## Output per candidate

| Field | Example |
| --- | --- |
| `name` | Priya Raman |
| `email` | priya.raman@examplemail.com |
| `phone` | (512) 555 0147 |
| `years_experience` | 5 |
| `skills_matched` | React, Node.js, JavaScript, REST APIs, Git |
| `skills_missing` | (none) |
| `score` | 0 to 100 |
| `verdict` | shortlist, maybe, or reject |
| `summary` | one-line rationale |

A Structured Output Parser enforces the shape, so a malformed model reply fails
the node rather than writing invalid data to the sheet.

## Role criteria

Set in the **Edit Fields** node. Change these to screen for a different role; no
other node needs touching.

```
job_title:        Software Engineer
must_have_skills: JavaScript, React, Node.js, REST APIs, Git
min_years:        3
nice_to_have:     TypeScript, AWS, Docker, CI/CD
```

## Sample results

Three resumes, one set of criteria, three different verdicts.

| Candidate | Years | Score | Verdict | Skills matched | Skills missing |
| --- | --- | --- | --- | --- | --- |
| Priya Raman | 5 | 100 | shortlist | JavaScript, React, Node.js, REST APIs, Git | (none) |
| George Marco | 7.5 | 65 | maybe | Node.js, REST APIs, Git | JavaScript, React |
| Kevin Walsh | 2.5 | 10 | reject | (none) | JavaScript, React, Node.js, REST APIs |

The three resumes are in `sample-resumes/`. Email them to yourself as PDF
attachments to reproduce the run.

Note that George is credited with 7.5 years rather than the 10 his resume spans.
The model counted relevant experience, not total time employed.

## Why criteria matching rather than keyword matching

Tested against three resumes:

**Strong fit.** 5 years, React and Node throughout, REST APIs, CI/CD, quantified
results on every bullet. Shortlisted.

**Clear reject.** Java and some Python, no React or Node, fragmented employment
with an unexplained gap, incomplete education. Rejected.

**The interesting one.** A senior engineer with 7.5 years of relevant experience, an M.Sc., and solid
open-source projects, whose stack is Java, Go, Swift, and Kotlin. Node appears
once, in a side project. React never.

Keyword matching shortlists him on "senior" and "full-stack". Criteria matching
flags him as a miss and names the gap: React and JavaScript absent. Strong
candidate, wrong role.

That distinction is the point of the workflow.

## Setup

**Requires:** n8n, a Google Gemini API key, Gmail OAuth credential, Google Sheets
OAuth credential.

1. Import `Resume_Screening_Workflow.json` into n8n
2. Reconnect the three credentials (Gmail, Google Sheets, Gemini)
3. Create a Google Sheet with these headers in row 1:

```
Date | Name | Email | Phone | Years | Score | Verdict | Skills Matched | Skills Missing | Summary
```

4. Point the three Google Sheets nodes at your sheet
5. Create four Gmail labels: one applied to every processed message (`screened`),
   plus one each for shortlist, maybe, and reject
6. Update the label IDs in the three **Add label** nodes. The IDs in this export
   belong to the original account and will not resolve on yours
7. Edit the criteria in **Edit Fields** for your role

## Known limitations

- **One resume per email.** The extract node reads `attachment_0` only. A message
  with two PDFs processes the first and silently ignores the rest.
- **Ten messages per run.** The Gmail node has a limit of 10. Larger batches need
  either a higher limit or a loop.
- **Label IDs are account-specific.** They are hardcoded and must be replaced
  after import, as noted in setup.
- **Three identical Sheets nodes.** One per verdict branch. Functionally correct,
  but they could merge into a single node after the Switch.
- **Manual trigger only.** A Gmail Trigger or Schedule Trigger would make it run
  unattended; the manual trigger is here for demonstration.
- **Text-layer PDFs only.** A scanned resume with no text layer extracts nothing.
  OCR would need to be added ahead of the extract node.
- **No handling for a model refusal.** If Gemini returns something the parser
  rejects, the run fails rather than routing the candidate for manual review.

## Notes

- Gmail query `-label:screened` is what prevents reprocessing. The label has to
  exist before the first run or the filter matches nothing.
- Rejected candidates are logged with their reason, not discarded. The audit trail
  matters more than the shortlist.
- Model: `models/gemini-3.5-flash`. Model names change; if the node returns a 404,
  check what is currently available for your key.
