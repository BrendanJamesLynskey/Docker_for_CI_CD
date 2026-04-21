# ⚙ Docker for CI/CD

An interactive Reveal.js presentation on Docker in CI/CD pipelines — GitHub Actions, GitLab CI, Jenkins, image tagging, layer caching, security scanning, and deployment strategies.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/Docker_for_CI_CD/)

## 📄 [Markdown Version](presentation.md)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Why Docker in CI/CD Pipelines | Consistent, reproducible environments at every stage |
| 02 | Docker-in-Docker vs Socket Binding | DinD isolation vs shared-daemon performance |
| 03 | GitHub Actions with Docker | Build, login, and push with official actions |
| 04 | GitLab CI with Docker | Docker executor and DinD services |
| 05 | Jenkins with Docker Agents | Ephemeral container-based build agents |
| 06 | Building Images in CI | Multi-stage builds, BuildKit, .dockerignore |
| 07 | Image Tagging Strategies | Git SHA, semver, branch, date-based tags |
| 08 | Layer Caching in CI | GHA cache, registry cache, inline cache |
| 09 | Running Tests in Containers | Tests in build stages and against running containers |
| 10 | Docker Compose for Integration Testing | Full stack spun up for integration tests |
| 11 | Container Registries in CI | GHCR, ECR, ACR, GAR, Docker Hub |
| 12 | Automated Security Scanning | Trivy, Snyk, Docker Scout in pipelines |
| 13 | Deployment Strategies | Rolling, blue-green, canary |
| 14 | GitOps with Docker Images | ArgoCD, Flux, Git as source of truth |
| 15 | CI/CD Pipeline Example Walkthrough | End-to-end GitHub Actions workflow |
| 16 | Performance Optimization Tips | Build speed, image size, runner tuning |
| 17 | Summary & Further Reading | Key takeaways and official docs |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## References

[Docker Build CI Documentation](https://docs.docker.com/build/ci/) · [GitHub Actions Docker Guide](https://docs.github.com/en/actions/publishing-packages/publishing-docker-images) · [GitLab CI Docker Builds](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html) · [Trivy Documentation](https://aquasecurity.github.io/trivy/)

## License

Educational use. Code examples provided as-is.
