# Automated CV Scanner
# ARCHITECTURE.md

## Technical Architecture & Design Decisions

---

## System Design Philosophy

The Automated CV Scanner is built on **event-driven orchestration** with **data redundancy** and **deterministic evaluation**. Every design decision prioritizes:

1. **Reliability**: Immediate logging prevents data loss
2. **Determinism**: JSON schema enforcement ensures reproducibility
3. **Scalability**: Stateless nodes enable parallel processing
4. **Maintainability**: Clear node responsibilities and data contracts

---

## Core Architectural Principles

### 1. Redundancy-First Logging

**Problem**: If downstream processing fails, we lose applicant data.

**Solution**: Immediate append to Google Sheets after form submission.

```
Application Form → (parallel paths)
  ├─→ Log Candidate Submission [append] ← IMMEDIATE PERSISTENCE
  └─→ Extract CV Text → [downstream processing]
```

Even if the workflow fails after CV extraction, the candidate's name and email are already logged.

### 2. Schema-Strict Output Validation

**Problem**: LLMs may generate malformed JSON; inconsistent field names break downstream logic.

**Solution**: LangChain Output Parser enforces strict schema before conditional branching.

```
AI Qualification (Gemini)
  ↓
JSON Output Parser
  ├─ Valid: → qualificationRate (number), explanation (string)
  └─ Invalid: → Reject and alert
```

**Benefits**:
- No silent failures
- Predictable data shapes for Sheets updates
- Auditable reasoning (explanation required)

### 3. Threshold-Based Deterministic Branching

**Problem**: Ad-hoc decision-making varies across runs.

**Solution**: Single, configurable threshold (0.75) enforces consistency.

```
qualificationRate >= 0.75 → Path 1 (Interview Invitation)
qualificationRate < 0.75  → Path 2 (Rejection)
```

**Rationale**:
- Legally defensible (same logic for all candidates)
- Easily adjustable without code changes
- Transparent to stakeholders

---

## Component Responsibilities & Data Contracts

### Layer 1: Ingestion

**Node**: Application Form (Form Trigger)
**Input**: User submission (name, email, PDF file)
**Output**: 
```json
{
  "Full Name": "Jane Doe",
  "Email": "jane@example.com",
  "Upload CV": "Jane_Doe_CV.pdf",
  "binary": { "... file binary": "..." }
}
```

**Constraints**:
- PDF only (`.pdf` validation)
- All fields required
- File size limit: 10 MB

---

### Layer 2: Data Persistence

**Node**: Log Candidate Submission (Google Sheets Append)
**Input**: Form output
**Output**: New row appended to Sheets
**Operation**: Append (idempotent but not upsert)

**Schema**:
```
| FullName | Email | Upload CV | submittedAt | formMode |
|----------|-------|-----------|-------------|----------|
| Jane Doe | jane@example.com | Jane_Doe_CV.pdf | 2025-01-15T... | ... |
```

**Why Append vs. Update?**
- Append is faster (no row lookup)
- Prevents data loss on partial failures
- Audit trail of all submissions

---

### Layer 3: Text Extraction

**Node**: Extract CV Text (Mistral OCR)
**Input**: PDF binary from Application Form
**Output**:
```json
{
  "text": "Jane Doe\n...\nProfessional Summary\nExperienced...",
  "pages": 2,
  "confidence": 0.95
}
```

**Error Handling**:
- **Timeout** (>30s): Log error, use rejection path
- **Invalid PDF**: Log error, send user feedback email
- **Low confidence** (<0.6): Flag in explanation, lower score

**Provider: Mistral**
- Rationale: Vision model accuracy > OCR-only tools for complex layouts
- Fallback: If unavailable, use alternative OCR (e.g., Azure CV Analyzer)

---

### Layer 4: Context Retrieval

**Node**: Get Job Description (Google Sheets Lookup)
**Input**: Job Title (from hardcoded prompt or dynamic lookup)
**Output**:
```json
{
  "JobTitle": "Senior Frontend Developer",
  "JobDescription": "5+ years React, TypeScript, ...",
  "CoreRequirements": ["Experience: 5+yr", "React: Expert", ...],
  "PreferredQualifications": ["Next.js", "Node.js", ...]
}
```

**Database**: Separate Google Sheet (`Job_Descriptions_DB`)

**Why External JD Source?**
- Decouples job specs from prompt (easy A/B testing)
- Supports multiple JD versions (junior, senior, contract roles)
- Audit trail of evaluation criteria changes

---

### Layer 5: AI Evaluation Engine

**Node**: AI Qualification (LangChain Chain + Google Gemini)
**Input**: 
```
- CV Text (from Mistral)
- Job Description (from Sheets lookup)
- Evaluation Prompt (structured prompt)
```

**Output** (before parsing):
```
{
  "qualificationRate": 0.82,
  "explanation": "Strong match: 5.5 years frontend, React 4yr, TypeScript expert, ..."
}
```

**Prompt Architecture**:

```
<goal>
Evaluate CV against Senior Frontend Developer job requirements.
Output: JSON with qualificationRate (0.0–1.0) and explanation.
</goal>

<context>
[Job requirements, core vs. preferred tiers]
</context>

<evaluation_logic>
Core Requirements (60% weight):
  - Each requirement: present/absent or scaled
  - Missing ANY core → max 0.6

Preferred Qualifications (40% weight):
  - Each adds +0.05 to base
  - Capped at 1.0
</evaluation_logic>

<output_format>
{
  "qualificationRate": <number 0.0-1.0>,
  "explanation": "<evidence from CV>"
}
</output_format>
```

**Model**: Google Gemini (2.0 Flash)
- **Why**: Fast, multimodal, structured output support
- **Cost**: ~$0.10 per 1M input tokens (efficient for CV text)
- **Latency**: ~2-5 seconds per candidate

---

### Layer 6: Output Validation

**Node**: JSON Output Parser (LangChain Structured Parser)
**Input**: Raw Gemini response (string)
**Schema**:
```json
{
  "type": "object",
  "properties": {
    "qualificationRate": { "type": "number", "min": 0, "max": 1 },
    "explanation": { "type": "string" }
  },
  "required": ["qualificationRate", "explanation"],
  "additionalProperties": false
}
```

**Validation Steps**:
1. JSON parse attempt
2. Type checking (number vs. string)
3. Range validation (0.0–1.0)
4. Field presence (no missing keys)
5. No extra keys allowed

**Error Handling**:
- **Parse Error**: Log to Sheets, mark as "ERROR", send alert
- **Type Mismatch**: Retry Gemini with stricter prompt
- **Out of Range**: Clamp to [0.0, 1.0], log warning

---

### Layer 7: Decision Logic

**Node**: If Condition
**Expression**:
```javascript
$json.output.qualificationRate >= 0.75
```

**Branches**:
- **True** (qualificationRate >= 0.75): Qualified path
- **False** (qualificationRate < 0.75): Rejected path

**Design Note**:
- Single condition (simplicity)
- Configurable threshold (flexibility)
- No nested logic (avoids bugs)

---

### Layer 8: Data Synchronization

**Node**: Add CV Analysis (Google Sheets Update via Email match)
**Input**: 
```json
{
  "Email": "jane@example.com",
  "QualificationRate": 0.82,
  "QualificationDescription": "..."
}
```

**Operation**: Update (upsert by Email)
**Schema Update**:
```
| Email | QualificationRate | QualificationDescription |
|-------|-------------------|-------------------------|
| jane@example.com | 0.82 | Strong match: 5.5y exp, React/TS expert, ... |
```

**Why Email as Matching Column?**
- Unique identifier (no duplicates)
- Aligns with candidate identity
- Enables manual lookup by recruiter

---

### Layer 9: Communication

**Nodes**: Gmail Nodes (Qualified & Rejected)

#### Qualified Candidates
```
To: {{ $('Application Form').last().json.Email }}
Subject: Congratulations! Interview Invitation
Body: 
  Dear {{ $json["Full Name"] }},
  
  [Personalized message with next steps]
  
  Best regards,
  Hiring Team
```

#### Rejected Candidates
```
To: {{ $('Application Form').last().json.Email }}
Subject: Thank You for Your Application
Body:
  Dear {{ $json["Full Name"] }},
  
  [Professional rejection with encouragement to reapply]
  
  Best regards,
  Hiring Team
```

**Configuration**:
- Sender: `hiring@company.com`
- SMTP credentials from Gmail OAuth
- Rate limiting: Max 100 emails/min per account

---

## Data Flow Diagram (Detailed)

```
┌─────────────────┐
│ Application     │
│ Form Trigger    │
└────────┬────────┘
         │ {name, email, pdf}
         ├─────────────────┐
         │                 │
         ↓                 ↓
    ┌─────────────┐  ┌──────────────┐
    │ Log         │  │ Extract CV   │
    │ Candidate   │  │ Text (OCR)   │
    │ (Append)    │  └────┬─────────┘
    └─────────────┘       │ {text}
                          ↓
                     ┌──────────────┐
                     │ Get Job      │
                     │ Description  │
                     └────┬─────────┘
                          │ {jd_text}
                          ↓
                     ┌─────────────────┐
                     │ AI              │
                     │ Qualification   │
                     │ (Gemini)        │
                     └────┬────────────┘
                          │ {json_str}
                          ↓
                     ┌─────────────────┐
                     │ JSON Parser     │
                     │ (Validate)      │
                     └────┬────────────┘
                          │ {validated}
                          ↓
                     ┌─────────────────┐
                     │ If Condition    │
                     │ (>= 0.75?)      │
                     └────┬────────────┘
                          │
            ┌─────────────┴──────────────┐
            │                            │
        (YES)                          (NO)
            ↓                            ↓
     ┌─────────────┐            ┌─────────────┐
     │ Update      │            │ Update      │
     │ Sheets      │            │ Sheets      │
     └─────┬───────┘            └──────┬──────┘
           ↓                           ↓
     ┌─────────────┐            ┌─────────────┐
     │ Send        │            │ Send        │
     │ Interview   │            │ Rejection   │
     │ Invitation  │            │ Email       │
     └─────────────┘            └─────────────┘
```

---

## Error Handling Strategy

| Failure Point | Detection | Mitigation | Fallback |
|---------------|-----------|-----------|----------|
| Form Submission Invalid | Form validation | User sees error message | N/A |
| Sheets Append Fails | Google API error | Retry 3×, exponential backoff | Alert admin |
| OCR Timeout | Mistral timeout | Log error, retry OCR | Skip extraction |
| Gemini Unavailable | API 503 | Queue and retry later | Reject candidate (0.5 score) |
| JSON Parse Fails | Parser error | Retry Gemini with stricter prompt | Mark as ERROR |
| Email Delivery Fails | Gmail API error | Retry 3×, log to Sheets | Manual follow-up needed |

**Error Workflow**: Separate n8n workflow catches all errors and sends alert email to admin.

---

## Performance Characteristics

| Operation | Latency | Throughput | Cost/Run |
|-----------|---------|-----------|----------|
| Form ingestion | <100ms | N/A | $0 |
| Sheets append | 500ms–1.5s | 100 rows/min | $0 |
| OCR extraction | 2–5s | 12 CVs/min | $0.01–0.05 |
| Gemini evaluation | 2–5s | 12 candidates/min | $0.10 |
| JSON parsing | <100ms | N/A | $0 |
| Sheets update | 500ms–1.5s | 100 updates/min | $0 |
| Email dispatch | 1–3s | 30 emails/min | $0 |

**End-to-End**: ~7–15 seconds per candidate

**Scaling**: Can handle ~500 applications/day within API quotas.

---

## Security Considerations

### Credential Management
- **Never commit** `.env` or credential files
- Use **n8n credential encryption** (key rotation recommended)
- **Least privilege**: Service accounts with minimal scopes
- **Audit logging**: All API calls logged to Sheets

### Data Privacy
- CVs stored only in Google Sheets (under your control)
- Gemini processes CV text (check Google's data retention policies)
- Email templates don't expose other candidates' data

### Access Control
- Workflow accessible only to authenticated n8n users
- Google Sheets shared only with hiring team
- Gmail account restricted to HR department

---

## Future Enhancements

1. **Batch Processing**: Queue multiple CVs, process in parallel
2. **Multi-Language Support**: Extend Mistral OCR to non-English CVs
3. **A/B Testing Framework**: Evaluate multiple Gemini prompts simultaneously
4. **Feedback Loop**: Collect hiring team feedback to improve scoring
5. **Integration with ATS**: Sync results to Lever, Greenhouse, Workable
6. **Resume Ranking**: Percentage-based ranking instead of binary pass/fail

---

**Version**: 1.0.0 | **Last Updated**: December 2025