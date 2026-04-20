# Docker + GitHub Actions validation guide

This guide shows how to run the same lint/test flow locally and in GitHub Actions using a Docker image.

## 1) Create a CI Dockerfile

Create `Dockerfile.ci` in the repository root:

```dockerfile
FROM python:3.11-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /workspace

# Configure Proton private package index (required for proton-core deps)
ARG GITLAB_TOKEN
ARG GITLAB_INSTANCE
ARG GROUP_ID
RUN python -m pip install --upgrade pip && \
    pip config set global.index-url \
    "https://__token__:${GITLAB_TOKEN}@${GITLAB_INSTANCE}/api/v4/groups/${GROUP_ID}/-/packages/pypi/simple"
```

## 2) Build the image locally

```bash
docker build \
  --build-arg GITLAB_TOKEN="$GITLAB_TOKEN" \
  --build-arg GITLAB_INSTANCE="$GITLAB_INSTANCE" \
  --build-arg GROUP_ID="$GROUP_ID" \
  -f ./Dockerfile.ci \
  -t proton-vpn-cli-ci:local \
  .
```

## 3) Run lint + tests locally in that container

```bash
docker run --rm \
  -v "$(pwd)":/workspace \
  -w /workspace \
  proton-vpn-cli-ci:local \
  bash -lc "pip install -r requirements.txt && python -m flake8 proton tests && python -m pytest"
```

## 4) Use the same image in GitHub Actions

Create `.github/workflows/validate.yml`:

```yaml
name: Validate

on:
  pull_request:
  push:
    branches: [stable]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    container:
      # Replace <owner> with your GitHub org/user that publishes the image.
      image: ghcr.io/<owner>/proton-vpn-cli-ci:latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure private package index at runtime
        # Keep credentials in Actions secrets instead of baking them into the image.
        run: |
          pip config set global.index-url \
            "https://__token__:${{ secrets.GITLAB_TOKEN }}@${{ secrets.GITLAB_INSTANCE }}/api/v4/groups/${{ secrets.GROUP_ID }}/-/packages/pypi/simple"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Lint
        run: python -m flake8 proton tests

      - name: Test
        run: python -m pytest
```

## 5) Notes

- End-to-end test execution requires private Proton packages (`proton-core`, `proton-vpn-api-core`, `proton-keyring-linux`, `proton-vpn-local-agent`, and dev extras).
- Keep credentials in GitHub Secrets only.
- If you already have a local checkout of installable `proton-core` and related repos, you can replace package-index install with editable installs (`pip install -e /path/to/repo`).
