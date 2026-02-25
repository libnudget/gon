# gon

A calm GitHub Actions workflow that gently reviews Dependabot pull requests and provides a clear, human-readable summary to support thoughtful merges.

gon is designed to stay quiet, helpful, and unobtrusive.

## Quick Start

Copy the workflow file to your repository:

```yaml
# .github/workflows/gon.yml
name: gon

on:
  pull_request:
    types: [opened, synchronize, reopened]
  issue_comment:
    types: [created, edited]

jobs:
  analyze-dependabot-pr:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: read

    steps:
      - name: Determine if should run
        id: should-run
        run: |
          if [ "${{ github.actor }}" = "dependabot[bot]" ]; then
            echo "RUN=true" >> $GITHUB_OUTPUT
            echo "PR_NUMBER=${{ github.event.pull_request.number }}" >> $GITHUB_OUTPUT
            echo "TRIGGER_USER=dependabot[bot]" >> $GITHUB_OUTPUT
          else
            echo "RUN=false" >> $GITHUB_OUTPUT
          fi

      - name: Get changed files
        if: steps.should-run.outputs.RUN == 'true'
        id: changed-files
        run: |
          PR_NUM="${{ steps.should-run.outputs.PR_NUMBER }}"

          PR_DATA=$(curl -s -H "Authorization: token ${{ github.token }}" \
            -H "Accept: application/vnd.github.v3+json" \
            "https://api.github.com/repos/${{ github.repository }}/pulls/$PR_NUM")

          echo "PR_TITLE=$(echo "$PR_DATA" | jq -r '.title')" >> $GITHUB_OUTPUT

          FILES_JSON=$(curl -s -H "Authorization: token ${{ github.token }}" \
            -H "Accept: application/vnd.github.v3+json" \
            "https://api.github.com/repos/${{ github.repository }}/pulls/$PR_NUM/files?per_page=100")

          FILES=$(echo "$FILES_JSON" | jq -r '.[].filename')

          DEP_PATTERNS="requirements|setup.py|pyproject.toml|Pipfile|package.json|yarn.lock|pnpm-lock.yaml|Cargo.toml|go.mod|go.sum|composer.json|pom.xml|\\.csproj$|Gemfile"

          DEP_FILES=""
          for file in $FILES; do
            if echo "$file" | grep -qiE "$DEP_PATTERNS"; then
              DEP_FILES="$DEP_FILES $file"
            fi
          done

          echo "CHANGED_FILES=$DEP_FILES" >> $GITHUB_OUTPUT

      - name: Analyze changes
        if: steps.should-run.outputs.RUN == 'true'
        id: analysis
        run: |
          PR_TITLE="${{ steps.changed-files.outputs.PR_TITLE }}"

          if [ -z "$CHANGED" ]; then
            echo "impact=low" >> $GITHUB_OUTPUT
            echo "status=No dependency files were modified." >> $GITHUB_OUTPUT
            exit 0
          fi

          echo "impact=medium" >> $GITHUB_OUTPUT

          if echo "$PR_TITLE" | grep -qi "security"; then
            echo "status=Security-related update. Review calmly before merging." >> $GITHUB_OUTPUT
          elif echo "$PR_TITLE" | grep -qiE "(major|breaking)"; then
            echo "impact=high" >> $GITHUB_OUTPUT
            echo "status=Major update detected. Testing is recommended." >> $GITHUB_OUTPUT
          elif echo "$PR_TITLE" | grep -qiE "(minor|patch|chore|deps|fix)"; then
            echo "impact=low" >> $GITHUB_OUTPUT
            echo "status=Minor update. Usually safe to merge." >> $GITHUB_OUTPUT
          else
            echo "status=Review the dependency changes before merging." >> $GITHUB_OUTPUT
          fi

      - name: Post summary comment
        if: steps.should-run.outputs.RUN == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            const impact = '${{ steps.analysis.outputs.impact }}';
            const status = '${{ steps.analysis.outputs.status }}';
            const files = '${{ steps.changed-files.outputs.CHANGED_FILES }}';
            const pr = Number('${{ steps.should-run.outputs.PR_NUMBER }}');
            const user = '${{ steps.should-run.outputs.TRIGGER_USER }}';

            const symbol = impact === 'high' ? '⚠️' : impact === 'medium' ? '◻️' : '✓';

            const body =
              `Hi @${user},\n\n` +
              `I took a calm look at this Dependabot pull request.\n\n` +
              `## gon ${symbol}\n\n` +
              `${status}\n\n` +
              `**Changed files:** ${files || 'None'}\n\n` +
              `---\n` +
              `_Reviewed quietly by gon_`;

            const comments = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: pr,
            });

            const existing = comments.data.find(c =>
              c.user.type === 'Bot' && c.body.includes('_Reviewed quietly by gon_')
            );

            if (existing) {
              await github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: existing.id,
                body,
              });
            } else {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: pr,
                body,
              });
            }

      - name: Skip
        if: steps.should-run.outputs.RUN != 'true'
        run: echo "gon stayed quiet."
```

## Features

- **Automatic triggering** — Runs on Dependabot PRs automatically
- **Comment triggers** — Use `/analyze`, `/run analysis`, or `/check pr` on any PR
- **Manual trigger** — Use `workflow_dispatch` to analyze any PR by number
- **Impact assessment** — Categorizes updates as low, medium, or high impact
- **Smart detection** — Identifies security, major, and minor updates from PR titles
- **Non-intrusive** — Stays quiet when not triggered, no noise from other PRs

## Triggers

| Event | Behavior |
|-------|----------|
| Dependabot PR opened/synced | Automatically analyzes |
| `/analyze` comment | Analyzes the PR |
| `workflow_dispatch` | Manually trigger with PR number |

## Impact Levels

| Level | Symbol | When |
|-------|--------|------|
| Low | ✓ | Minor, patch, chore, deps, fix updates |
| Medium | ◻️ | Regular dependency updates |
| High | ⚠️ | Major/breaking updates, security fixes |

## Viewing the Workflow Locally

If you want to review or adjust the workflow before opening a pull request, simply inspect and edit the workflow file locally:

```bash
.github/workflows/gon.yml
```

No local server, build step, or additional tooling is required.

**Note:** This repository intentionally contains only workflow logic.
Please avoid committing generated files or unrelated artifacts in pull requests.

---

# Contributing

gon is an Open Source project licensed under the permissible MIT License.

Contributions are welcome.
Please open an issue to discuss a change, followed by a pull request implementing the improvement.

---

### Philosophy

Automation should assist quietly.

gon exists to reduce noise and provide calm guidance without blocking, enforcing, or interrupting existing workflows.
