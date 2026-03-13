---
name: github-pr-review
description: Automated pull request review workflow with inline comments, approval/request-changes decisions, and summaries.
---

# GitHub PR Review Skill

Automated workflow for reviewing pull requests with detailed feedback, code analysis, and approval decisions.

## Overview

This skill automates the GitHub pull request review process by:
- Analyzing code changes
- Identifying potential issues and improvements
- Providing inline comments with specific suggestions
- Making approval or request-changes decisions
- Generating a review summary

## Invocation

```
/github-pr-review <PR_URL_or_NUMBER>
```

## Workflow

### Step 1 — Fetch PR Details
Retrieve PR information, changed files, and diffs.

### Step 2 — Analyze Changes
Review code for:
- Correctness and logic errors
- Performance implications
- Security vulnerabilities
- Style and best practices

### Step 3 — Generate Comments
Create inline comments on specific lines with actionable feedback.

### Step 4 — Make Decision
Provide approval, request changes, or comment-only review.

### Step 5 — Summary
Report findings and next steps.

## Configuration

Customize review criteria by modifying this skill's parameters:
- Comment style (concise vs. detailed)
- Approval threshold
- Priority issues to highlight
- Language/framework-specific rules

## Notes

This is a template skill. Customize the analysis criteria and decision logic to match your project's standards and review process.
