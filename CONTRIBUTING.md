# Automated CV Scanner
# CONTRIBUTING.md

## Code of Conduct

This project adheres to a Code of Conduct aimed at fostering an open, welcoming, and inclusive community. All contributors are expected to uphold these standards of respectful and professional interaction.

---

## How to Contribute

### Reporting Bugs

Found a bug? We'd love to know. Please:

1. **Check existing issues** to avoid duplicates
2. **Create a new issue** with:
   - Clear title and description
   - Steps to reproduce
   - Expected vs. actual behavior
   - n8n version, workflow export (sanitized of credentials)
   - Logs or error messages

**Template:**
```
**Description**: [Brief description of the bug]

**Steps to Reproduce**:
1. ...
2. ...

**Expected Behavior**: [What should happen]
**Actual Behavior**: [What actually happens]

**Environment**:
- n8n Version: X.X.X
- Browser/Client: ...
- Error Logs: [Paste relevant logs]
```

### Suggesting Enhancements

Have an idea to improve the project?

1. **Open an issue** with the label `enhancement`
2. **Describe the use case** and expected benefit
3. **Provide examples** or mockups if applicable

**Template:**
```
**Feature Request**: [One-line summary]

**Problem It Solves**: [Describe the current pain point]
**Proposed Solution**: [How should it work?]
**Example Use Case**: [Real-world scenario]
```

### Submitting Code

#### Step 1: Fork & Clone

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/automated-cv-scanner.git
cd automated-cv-scanner
git remote add upstream https://github.com/ORIGINAL_OWNER/automated-cv-scanner.git
```

#### Step 2: Create a Feature Branch

```bash
git checkout -b feature/your-feature-name
# or
git checkout -b bugfix/issue-number
```

#### Step 3: Make Your Changes

- Follow the existing code style and structure
- Update documentation if needed
- Test thoroughly before submitting

#### Step 4: Commit with Clear Messages

```bash
git commit -m "feat: add multi-JD support for batch processing"
# or
git commit -m "fix: resolve JSON parser schema validation error"
```

**Commit conventions**:
- `feat:` New feature
- `fix:` Bug fix
- `docs:` Documentation update
- `refactor:` Code refactoring without feature change
- `test:` Tests or test infrastructure
- `chore:` Build, dependencies, or tooling

#### Step 5: Push & Create Pull Request

```bash
git push origin feature/your-feature-name
```

Then on GitHub:
1. Open a Pull Request against `main`
2. Fill in the PR template
3. Link any related issues
4. Request review

**PR Template:**
```
## Description
[Brief description of changes]

## Related Issue
Fixes #[issue_number]

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Documentation update

## Checklist
- [ ] Code follows style guidelines
- [ ] Comments added for complex logic
- [ ] Documentation updated
- [ ] Tested with sample workflows
- [ ] No new warnings or errors

## Testing Evidence
[Describe how you tested, include screenshots or logs]
```

### Workflow-Level Contributions

Improving the n8n workflow itself?

1. **Export your modified workflow** as JSON
2. **Document changes** clearly (which nodes changed, why)
3. **Test end-to-end** with sample CVs
4. **Submit PR** with:
   - Updated `its-worked.json`
   - Changelog entry in `docs/CHANGELOG.md`
   - Updated architecture diagram if relevant

---

## Development Guidelines

### Workflow Modifications

- **Preserve backward compatibility** when possible
- **Document node parameter changes** in API_REFERENCE.md
- **Test credential switching** (ensure workflow doesn't break with different credentials)
- **Validate JSON schema** changes with the Output Parser

### Documentation

- Use **clear, professional English**
- Include **code examples** where helpful
- **Update table of contents** for longer docs
- Link to relevant **n8n or Google Cloud docs**

### Testing Your Changes

#### Test Locally

1. Import the modified workflow into your n8n instance
2. Create a test Google Sheet and Gmail alias
3. Upload a sample CV and verify:
   - Correct extraction
   - Proper AI scoring
   - Email delivery

#### Test with Examples

We provide sample CVs in `examples/sample_cvs/`:
- `strong_candidate.pdf` → Should score 0.80+
- `borderline_candidate.pdf` → Should score 0.60–0.75
- `poor_fit_candidate.pdf` → Should score <0.60

---

## Project Structure Overview

```
automated-cv-scanner/
├── its-worked.json             # Core workflow export
├── docs/
│   ├── SETUP.md                # Installation guide
│   ├── ARCHITECTURE.md         # Technical details
│   └── TROUBLESHOOTING.md      # Common issues
├── examples/                   # Sample data & exports
├── templates/                  # Configuration templates
└── CONTRIBUTING.md             # This file
```

---

## Pull Request Review Process

1. **Automated Checks**: GitHub Actions validates formatting and links
2. **Maintainer Review**: Code review for quality, security, and alignment
3. **Community Feedback**: Other contributors may suggest improvements
4. **Approval & Merge**: Approved PRs merged by maintainers

**Timeline**: Most PRs reviewed within 48–72 hours.

---

## Code of Conduct

This project is committed to providing a welcoming and inspiring community for all. We expect participants to:

- Be respectful and inclusive
- Welcome feedback and different perspectives
- Report violations privately to maintainers

---

## Recognition

Contributors are recognized in:
- GitHub contributors page
- Project CHANGELOG
- Release notes (for significant contributions)

Thank you for helping make recruitment automation more accessible! 🚀

---

**Last Updated**: December 2025