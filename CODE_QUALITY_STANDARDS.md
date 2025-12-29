# Automated CV Scanner
# CODE_QUALITY_STANDARDS.md

## Code Quality & Review Standards

---

## Workflow Quality Checklist

When submitting workflow modifications, ensure:

### Structural Integrity
- [ ] All nodes have descriptive names (not "Node 1", "Node 2")
- [ ] Comments explain complex logic in sticky notes
- [ ] Connections are explicitly mapped in `connections` object
- [ ] No orphaned or unreachable nodes
- [ ] Error handling is present for all API calls

### Data Contracts
- [ ] Input/output schemas documented for each node
- [ ] No unvalidated data passed to downstream nodes
- [ ] JSON parser enforces schema for all LLM outputs
- [ ] Type coercion is explicit (no silent string→number conversions)
- [ ] Null/undefined values handled gracefully

### Security
- [ ] No credentials hardcoded in node parameters
- [ ] All API keys reference credential objects
- [ ] Sensitive data (emails, SSNs) not logged to Sheets without encryption
- [ ] File uploads validated (size, type, content)
- [ ] Rate limiting respected for all third-party APIs

### Performance
- [ ] No nested loops or recursive processing
- [ ] Conditional branches minimize redundant computation
- [ ] Parallel paths used where applicable (e.g., append + extract simultaneously)
- [ ] Retry logic includes exponential backoff
- [ ] Timeout values are reasonable (no 0, no >60s without justification)

### Maintainability
- [ ] Node names reflect their purpose
- [ ] Complex prompts documented in `docs/PROMPTS.md`
- [ ] Configuration values extracted to `.env`
- [ ] Test workflows provided for new features
- [ ] CHANGELOG.md updated with version bump

---

## LLM Prompt Standards

### Prompt Structure
Prompts must follow this format:

```
<goal>
One-sentence objective
</goal>

<context>
[Structured data: job reqs, CV, evaluation criteria]
</context>

<instructions>
[Step-by-step reasoning]
</instructions>

<output_format>
{
  "field1": "<description>",
  "field2": "<description>"
}
</output_format>
```

### Prompt Versioning
- Store major prompt versions in `docs/PROMPTS.md`
- Document changes in CHANGELOG
- A/B test before rolling out to production

### Example: Well-Structured Prompt

```
<goal>
Evaluate a candidate's CV against Senior Frontend Developer job requirements.
Output a JSON object with a qualification score (0.0–1.0) and evidence-based explanation.
</goal>

<context>
<job_requirements>
- Required: 5+ years frontend experience
- Required: React expertise
- Preferred: Next.js experience
</job_requirements>

<candidate_cv>
{{ $('Extract CV Text').last().output.text }}
</candidate_cv>
</context>

<evaluation_logic>
1. Check for each required skill; missing any = max 0.6
2. For each preferred, add +0.05 (if met)
3. Provide point-by-point justification
</evaluation_logic>

<output_format>
{
  "qualificationRate": <number 0.0-1.0>,
  "explanation": "<point-by-point analysis citing CV evidence>"
}
</output_format>
```

---

## Testing Standards

### Unit Testing (Nodes)

For each modified node:
1. **Test valid input** → Verify correct output
2. **Test edge cases** → Empty strings, null values, extreme ranges
3. **Test error conditions** → Invalid JSON, API timeouts, malformed input

### Integration Testing

Before merging:
1. **End-to-end test** with sample CV
2. **Verify Sheets output** has correct schema
3. **Confirm email delivery** to test inbox
4. **Check error handling** by introducing intentional failures

### Test Data

Provide in `examples/sample_cvs/`:
- `strong_candidate.pdf` → Expected score: 0.80+
- `borderline_candidate.pdf` → Expected score: 0.70–0.75
- `poor_fit_candidate.pdf` → Expected score: <0.60

---

## Documentation Standards

### Node-Level Documentation

Each node should have:
- **Purpose**: One-sentence description
- **Input**: Expected schema and example
- **Output**: Returned schema and example
- **Configuration**: Required parameters and their values
- **Error Handling**: What happens if this node fails

**Example**:
```markdown
## Extract CV Text

**Purpose**: Converts PDF file to plaintext using Mistral OCR.

**Input**:
- Binary PDF file from Application Form
- Max file size: 10 MB

**Output**:
```json
{
  "text": "Jane Doe...",
  "pages": 2,
  "confidence": 0.95
}
```

**Configuration**:
- API: Mistral OCR (latest model)
- Timeout: 30 seconds

**Error Handling**:
- If timeout: Log error, use rejection path
- If invalid PDF: Send user feedback email
```

### Prompt Documentation

Complex prompts should be documented in `docs/PROMPTS.md`:

```markdown
## Senior Frontend Developer Evaluation Prompt

**Version**: 1.0.0
**Last Updated**: 2025-12-15
**Author**: Lucas Peyrin

**Purpose**: Evaluate CVs against Senior Frontend Developer job description.

**Structure**: 
- Goal statement
- Context (job reqs, CV)
- Evaluation logic
- Output schema

**Example Input/Output**:
[Include actual example]

**Tuning Notes**:
- Threshold of 0.75 chosen after testing 500+ CVs
- Weights: 60% core, 40% preferred
- Not suitable for non-technical roles

**Future Improvements**:
- Support multiple evaluation rubrics
- Collect hiring feedback to refine scoring
```

---

## Code Review Criteria

### For PRs Modifying Workflow JSON

**Reviewers should verify**:

1. **Correctness**
   - Does the change accomplish its stated goal?
   - Are data transformations accurate?
   - Is the schema valid for downstream nodes?

2. **Security**
   - No hardcoded credentials?
   - Sensitive data not exposed in logs?
   - File uploads validated?

3. **Performance**
   - No new bottlenecks?
   - Retry logic sensible?
   - API quotas not exceeded?

4. **Maintainability**
   - Code/workflow is clear and self-documenting?
   - Complex logic has comments?
   - Updated relevant documentation?

5. **Testing**
   - Includes test data?
   - End-to-end test completed?
   - Edge cases considered?

### Review Template

```markdown
## Code Review: [PR Title]

### ✅ Strengths
- [Positive observation]
- [Positive observation]

### ⚠️ Questions
- [Question about design]
- [Clarification needed]

### 🔧 Suggestions
- [Non-blocking improvement]
- [Optional refactor]

### 🚫 Blockers
- [ ] Security issue found
- [ ] Data loss risk
- [ ] Performance degradation

### ✓ Approval
- [x] Code quality acceptable
- [x] Testing adequate
- [x] Documentation updated
```

---

## Performance Benchmarks

Establish baseline metrics:

| Metric | Target | Current |
|--------|--------|---------|
| CV extraction time | <5s | 3.2s |
| Gemini evaluation time | <5s | 4.1s |
| End-to-end latency | <15s | 11.3s |
| Sheets write latency | <2s | 1.1s |
| Email delivery | <3s | 1.8s |
| Success rate | >99% | 99.4% |

Monitor these metrics in production; alert if they exceed thresholds.

---

## Version Control Strategy

### Branch Naming
- `feature/description` – New feature
- `bugfix/issue-number` – Bug fix
- `docs/topic` – Documentation only
- `refactor/description` – Code/workflow cleanup

### Commit Conventions
```
feat: add support for multi-language CV analysis
fix: resolve JSON parser schema validation error
docs: add troubleshooting guide for Mistral API
refactor: simplify If condition logic
test: add sample CV for edge case testing
```

### Release Process
1. Merge PR to `main`
2. Create release tag (`v1.1.0`)
3. Update CHANGELOG.md
4. Deploy to production

---

## Monitoring & Observability

### Metrics to Track
- Workflow execution count (daily)
- Success/failure rate (%)
- Average latency per stage
- API error rate (by service)
- Candidate qualification distribution

### Alerts
- Workflow failure rate >2%
- API timeout errors >5/hour
- Sheets quota exceeded
- Email bounce rate >5%

### Logs
All errors should be logged to a dedicated "Errors" sheet with:
- Timestamp
- Error type
- Node that failed
- Input data (sanitized)
- Resolution status

---

**Version**: 1.0.0 | **Last Updated**: December 2025