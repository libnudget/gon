# gon

This repository contains the source-code behind **gon**, a calm GitHub Actions workflow that gently reviews Dependabot pull requests and provides a clear, human-readable summary to support thoughtful merges.

gon is designed to stay quiet, helpful, and unobtrusive.

## Viewing the Workflow Locally

gon runs entirely on **GitHub Actions** and does not generate a website or static documentation.

If you want to review or adjust the workflow before opening a pull request, simply inspect and edit the workflow file locally:

```bash
.github/workflows/gon.yml
```

No local server, build step, or additional tooling is required.

**Note:** This repository intentionally contains only workflow logic.
Please avoid committing generated files or unrelated artifacts in pull requests.

# Contributing

gon is an Open Source project licensed under the permissible MIT License.

Contributions are welcome.
Please open an issue to discuss a change, followed by a pull request implementing the improvement.

---

### Philosophy

Automation should assist quietly.

gon exists to reduce noise and provide calm guidance without blocking, enforcing, or interrupting existing workflows.
