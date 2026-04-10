# use

A collection of setup guides, reusable GitHub Actions workflows, and configuration references for development tools.

## Contents

- [terminal/ghostty.md](terminal/ghostty.md) — Complete guide to install and configure Ghostty terminal with Powerlevel10k, autosuggestions, syntax highlighting, and the Catppuccin theme. Covers macOS, Linux, and Windows.

## GitHub Actions Workflows

Reusable CI/CD workflow templates located in [`.github/workflows/`](.github/workflows/).

| Workflow | File | Description |
|----------|------|-------------|
| **Test Build** | [`test-build.yml`](.github/workflows/test-build.yml) | Builds a TypeScript project using pnpm on every push |
| **Unit Tests** | [`unit-test.yml`](.github/workflows/unit-test.yml) | Runs unit tests for JS/TS projects across Node 18, 20, 22 |
| **Auto Release** | [`release.yml`](.github/workflows/release.yml) | Creates a GitHub Release automatically when a `v*.*.*` tag is pushed |
| **Deploy to EC2** | [`push-ec2.yml`](.github/workflows/push-ec2.yml) | Deploys to EC2 via SSH on push to `live` or `dev` branches |
| **Deploy to ECS** | [`push-ecs.yml`](.github/workflows/push-ecs.yml) | Builds a Docker image and pushes to AWS ECR on push to `live` |
| **S3 Backup** | [`push-s3.yml`](.github/workflows/push-s3.yml) | Zips the project and uploads to S3 on push to `live` |
| **Discord Notify** | [`notify.discord.yml`](.github/workflows/notify.discord.yml) | Sends a Discord webhook notification on every push |

## License

See [LICENSE](LICENSE).
