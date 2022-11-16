# Container build process

## Requirements

- Container scan failed (how to progress even when there are CRITICAL
  vulnerabilities in the container)
  - Set `container_image_vulnerability_checks=false`
- I want to build test container (with specific tag like
  `myc-hello-kubernetes:5.1.2-test1` with expiration date)
- I want to build test container for specific PR with expiration date
- I want to build only `latest` tag manually?
- `latest` tag should be built only form `main` branch...
- Newly released images must create/update following tags in container
  repository (example):
  - `1`
  - `1.2`
  - `1.2.0`
  - `latest`
- Container image should be scanned for "Critical" vulnerabilities
- Image should be signed ([cosign](https://github.com/sigstore/cosign))
- SBOM should be generated

## Possible build examples

- Tag "main" branch

  ```bash
  TAG="7.0.16"

  git tag "v${TAG}-beta.0" && git push origin "v${TAG}-beta.0"
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"

  git tag "v${TAG}-beta.1" && git push origin "v${TAG}-beta.1"
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"

  git tag "v${TAG}-rc.0" && git push origin "v${TAG}-rc.0"
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"

  git tag "v${TAG}-rc.1" && git push origin "v${TAG}-rc.1"
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"

  git tag "v${TAG}" && git push origin "v${TAG}"
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  ```

- Run build process form "main" branch - expires in 356 days (by default)

  ```bash
  gh workflow run container-build.yml -f container_registry_push=true -f container_image_vulnerability_checks=false -f container_image_expires_after=365
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  ```

- Run build process form "fix" branch - expires in 30 days (by default)

  ```bash
  git checkout -b fix && git push
  gh workflow run container-build.yml -f container_registry_push=true -f container_image_vulnerability_checks=false --ref=fix
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  git checkout main && git branch -d fix && git push origin -d fix
  ```

- Run build process form "fix2" branch form PR - expires in 30 days (by default)

  ```bash
  git checkout -b fix2
  date > date
  git add date && git commit -m "date" && git push
  PR_URL=$(gh pr create --fill)
  gh workflow run container-build.yml -f container_registry_push=true -f container_image_vulnerability_checks=false --ref=fix2
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  date > date
  git add date && git commit -m "date2" && git push
  gh workflow run container-build.yml -f container_registry_push=true -f container_image_vulnerability_checks=false --ref=fix2
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  gh pr close --delete-branch "${PR_URL}"
  ```

- Scheduled build - expires in 356 days
