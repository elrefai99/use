# Repository Guidelines

## Project Structure & Module Organization

This repository contains reusable development guidance and CI/CD templates rather than a single application:

- `.github/workflows/` contains reusable GitHub Actions workflows for checks, Docker Hub, EC2, ECS, S3, releases, and Discord notifications.
- `Skills/` contains agent guidance, including backend architecture, code review, issue fixing, message queues, and unit testing.
- `terminal/` contains platform-oriented setup documentation, such as `terminal/ghostty.md`.
- `README.md` documents the available guides and workflows.

Application source, tests, and assets are supplied by projects that consume these templates; they are not currently present here.

## Build, Test, and Development Commands

There is no local application build command for this repository. The reusable TypeScript check workflow expects the consuming project to provide:

```bash
pnpm install --frozen-lockfile
pnpm lint
pnpm test
pnpm build
```

Docker workflows build from the consuming repository root and use `PROJECT_NAME` and `github.sha` for image tags. Validate workflow changes with GitHub Actions syntax tooling when available, then review the affected workflow dependencies and secrets.

## Coding Style & Naming Conventions

- Use two-space indentation in YAML and preserve valid GitHub Actions expression syntax (`${{ ... }}`).
- Name workflow files with lowercase kebab-case, for example `push-docker-hub.yml`.
- Use descriptive job and step IDs; `needs` must reference job IDs, not display names.
- Keep Markdown headings hierarchical and use fenced code blocks for commands.

## Testing Guidelines

No repository test framework or coverage threshold is defined. For workflow changes, check YAML structure, reusable-workflow inputs, job ordering, and required secrets. For consuming TypeScript projects, run the four `pnpm` commands above and keep tests alongside that project’s established conventions.

## Commit & Pull Request Guidelines

Recent commits use Conventional Commit-style prefixes such as `feat:`, `fix:`, `chore:`, and `docs:`. Use a concise imperative subject, for example `fix: correct reusable workflow dependency`.

Pull requests should explain the workflow or documentation impact, identify changed files, mention required repository variables/secrets, and include validation results. Add screenshots only when changing rendered documentation or visual configuration.

## Security & Configuration

Never commit tokens, SSH keys, AWS credentials, or Discord webhooks. Configure secrets through GitHub repository or environment settings, and use least-privilege permissions for workflow jobs.

## Permissions

Global rule:

- Ask the user first before making any code change.
- Show the intended change for review when possible.
- Wait for the user to accept or reject the change before editing project code.
- Always keep the changes in the code to be as simple as possible, straight to the point, with no comments added

### Allow

- Read tracked project files needed for the task.
- Read source code under `src/`, configuration under `config/`, and docs such as `README.md`, `CLAUDE.md`, and this file.
- Create new source or documentation files when they are required for the requested change.
- Edit application code, route files, models, middleware, utilities, tests, and markdown documentation.
- Update `package.json` when the task explicitly requires script or dependency changes.
- Run safe repo-local commands such as `rg`, `ls`, `sed`, `git status`, `pnpm lint`, and `pnpm test`.

### Deny

- Do not read, print, or copy secrets from `.env`, `.env.dev`, or any credential file.
- Do not modify `.env`, `.env.dev`, or other secret-bearing files unless the user explicitly asks.
- Do not modify `node_modules/`, generated caches, or log files.
- Do not change `pnpm-lock.yaml` unless dependency work is part of the task.
- Do not delete files, rename major directories, or rewrite large parts of the codebase without explicit approval.
- Do not run destructive git or shell commands such as `git reset --hard`, `git checkout --`, or broad `rm` operations.
- Do not alter deployment/infrastructure files (`Dockerfile`, `docker-compose.yml`, `ecosystem.config.cjs`) unless the task explicitly requires it.
