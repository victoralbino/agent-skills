---
name: code-review
description: Code review with actionable recommendations. Use when the user asks for code review, /code-review, or quality analysis of code changes. Analyzes architecture, security, performance, conventions, and test coverage. Don't use for reviewing documentation-only changes, non-code files, or when the user asks to implement changes without reviewing first.
argument-hint: "[PR number]"
---

# Code Review

Follow these steps to conduct a thorough code review:

## 1. Gather Context

- If a PR number is provided in the args, run `gh pr view <number>` and `gh pr diff <number>`
- If no PR number is provided, run `gh pr list` to check for open PRs. If none, review local uncommitted changes using `git diff` (staged + unstaged) and `git status`
- Read all new and modified files to understand the full scope of changes

## 2. Analyze Changes

Provide a thorough code review covering:

- **Overview**: What the changes do (1-2 sentences)
- **Architecture**: Does it follow project conventions and domain architecture?
- **Code Quality**: Clean code, naming, duplication, separation of concerns
- **Security**: Injection, auth, data exposure, OWASP concerns
- **Performance**: N+1 queries, unnecessary operations, missing indexes
- **Test Coverage**: Are all paths tested? Missing edge cases?
- **Potential Risks**: Breaking changes, side effects, race conditions

## 3. Categorize Findings

Organize issues by severity:

| Severity | Meaning |
|----------|---------|
| **Critical** | Must fix — bugs, security vulnerabilities, data loss risk |
| **Important** | Should fix — architecture violations, missing validation, logic gaps |
| **Minor** | Nice to fix — code style, duplication, missing tests for edge cases |

For each finding, include:
- File and line reference
- Clear description of the issue
- Specific recommendation for how to fix it

## 4. Decision Round with AskUserQuestion

After presenting the review, **use the `AskUserQuestion` tool** to let the user decide which items to fix. This is a critical step — do not skip it.

### How to structure the questions:

- Group related findings into questions (max 4 questions per AskUserQuestion call)
- For each question, present the options with your **recommendation clearly marked** using "(Recommended)" suffix on the label
- Use `multiSelect: true` when presenting a list of independent fixes the user can pick from
- Use `multiSelect: false` when presenting mutually exclusive approaches to solve a single issue
- Include a clear description for each option explaining the tradeoff

### Example structure:

```
Question 1 (single-select): "The Action throws ValidationException from the Domain layer. Which approach do you prefer?"
  - "Move to Request (Recommended)" — keeps domain clean
  - "Domain exception" — more verbose but explicit
  - "Keep as-is" — pragmatic, works fine

Question 2 (multi-select): "Which minor fixes do you want me to apply?"
  - "Remove redundant index" — already created by foreign key
  - "Add DB transaction" — consistency with sibling action
  - "Extract duplicated method" — DRY principle
  - "Add missing test" — covers untested branch
```

### Rules for the decision round:

- Always present your recommendation — never leave the user without guidance
- Be concise in descriptions — the review already explained the details
- After the user answers, implement only what was approved
- If all findings are Critical, fix them immediately without asking (they are non-negotiable)

## 5. Apply Fixes

After the user decides:
- Implement the approved changes
- Run affected tests to verify
- Run the formatter (`php vendor/bin/pint --dirty --format agent`)
- Report what was done
