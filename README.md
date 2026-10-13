# Jenkins Pipeline

A declarative `Jenkinsfile` that lints, tests, builds, smoke-tests, and publishes a small Flask service
as a container image. GitHub Actions validates the `Jenkinsfile` itself on every push by running it through
Jenkins' own declarative linter, so a syntax error can't reach `main` unnoticed.

```
checkout ─► Prepare ─► Lint & test ─► Build image ─► Smoke test ─► Push image (main only)
              │            │                │             │               │
        short SHA tag  docker build   docker build   run + probe     docker login/push
                       --target test                 /health         ghcr.io (:sha, :latest)
```

## Stages

| Stage | What it does |
|---|---|
| Prepare | Derives the image tag from the short commit SHA |
| Lint & test | `docker build --target test .` runs `ruff` and `pytest` inside the Dockerfile's `test` stage |
| Build image | Builds the slim runtime image and bakes the SHA in as `APP_VERSION` |
| Smoke test | Starts the container and probes `/health` from inside it (works even when Jenkins itself runs in Docker) |
| Push image | `main` only. Logs in with the `ghcr-credentials` Jenkins credential and pushes `:<sha>` and `:latest` |

Design choices:
- **Tests live in the Dockerfile**, so the Jenkinsfile needs only the Docker CLI on the agent and local runs match CI exactly.
- **Secrets are never interpolated by Groovy.** The push step uses single-quoted shell with `withCredentials`, and the token goes through `--password-stdin`.
- **Cleanup always runs.** The `post { always }` block removes the smoke-test container and logs out of the registry.
- **Guardrails:** build timeout, no concurrent builds, and log rotation.

## Validation in CI (`.github/workflows/validate.yml`)

| Job | What it checks |
|---|---|
| `app` | Dockerfile `test` stage (lint + tests) and the runtime image build |
| `jenkinsfile` | Builds the Jenkins image in `jenkins/`, starts it, and posts the `Jenkinsfile` to `/pipeline-model-converter/validate` |

## Try it locally

```bash
docker compose up --build        # Jenkins at http://localhost:8080
docker compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Then in Jenkins:
1. **Manage Jenkins → Credentials:** add a *Username with password* credential with ID `ghcr-credentials`
   (your GitHub username and a personal access token with `write:packages`).
2. **New Item → Pipeline** (or Multibranch Pipeline), choose *Pipeline script from SCM*,
   point it at this repository, and keep the script path `Jenkinsfile`.
3. Build. Change `IMAGE` in the `Jenkinsfile` to your own registry path first.

> The compose file runs Jenkins as root with the host Docker socket mounted. That's convenient for a demo and
> unsafe for production; use dedicated build agents there.

## Run the app without Jenkins

```bash
pip install -r requirements-dev.txt
ruff check . && pytest -q
docker build -t pipeline-demo . && docker run --rm -p 8000:8000 pipeline-demo
```

## Next steps

- Add a deploy stage gated by an `input` approval step.
- Move shared logic into a Jenkins Shared Library.
- Add image scanning (e.g. Trivy) before the push stage.
