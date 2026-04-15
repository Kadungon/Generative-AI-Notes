.github/prompts/pr-review.prompt.md

---
mode: ask
description: Review PR with category scoring
---

Review this pull request.

Score these categories from 1 to 10:
- code_quality
- security
- testing
- readability
- maintainability

Rules:
- If every score >= 7, decision = APPROVE
- Otherwise decision = REQUEST_CHANGES
- Mention short reasons for each category
- Return only JSON

Example format:

{
  "decision": "APPROVE",
  "scores": {
    "code_quality": 8,
    "security": 9,
    "testing": 7,
    "readability": 8,
    "maintainability": 7
  },
  "reasons": {
    "code_quality": "Good structure",
    "security": "No major issues",
    "testing": "Adequate coverage",
    "readability": "Easy to understand",
    "maintainability": "Low duplication"
  }
}


.github/workflows/auto-ai-review.yml


name: Auto AI Review Approval

on:
  issue_comment:
    types: [created]
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  trigger-copilot-review:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest

    steps:
      - name: Ask Copilot for structured review
        uses: actions/github-script@v7
        with:
          script: |
            const body = `
            @copilot /pr-review
            `

            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.payload.pull_request.number,
              body
            })

  auto-approve:
    if: github.event_name == 'issue_comment'
    runs-on: ubuntu-latest

    steps:
      - name: Read Copilot review
        id: check
        run: |
          python << 'EOF'
          import json
          import os
          import re

          body = """${{ github.event.comment.body }}"""

          approved = False

          scores = re.findall(r':\s*(\d+)', body)

          if scores:
              approved = all(int(score) >= 7 for score in scores)

          with open(os.environ["GITHUB_OUTPUT"], "a") as f:
              f.write(f"approved={'true' if approved else 'false'}\n")
          EOF

      - name: Approve PR
        if: steps.check.outputs.approved == 'true'
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const prNumber = context.payload.issue.number

            await github.rest.pulls.createReview({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: prNumber,
              event: 'APPROVE',
              body: 'AI auto-approved because all category scores are >= 7.'
            })

      - name: Add label
        if: steps.check.outputs.approved == 'true'
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            await github.rest.issues.addLabels({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.payload.issue.number,
              labels: ['ai-approved']
            })
