# Container build process

The source code was taken from: <https://github.com/paulbouwer/hello-kubernetes>

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

## Local tests

Build commands:

```bash
docker build -f ./src/app/Dockerfile -t myc-hello-kubernetes:latest src/app
docker build -f ./src/app/Dockerfile-node-18-alpine -t myc-hello-kubernetes:latest-alpine src/app
docker build -f ./src/app/Dockerfile-node-18-debian-slim -t myc-hello-kubernetes:latest-debian-slim src/app
docker build -f ./src/app/Dockerfile-nodejs18-distroless -t myc-hello-kubernetes:latest-distroless src/app
docker build -f ./src/app/Dockerfile-nodejs-16-ubi -t myc-hello-kubernetes:latest-ubi src/app
```

Run commands:

```bash
docker run -p 8080:8080 --rm myc-hello-kubernetes:latest
docker run -p 8080:8080 --rm myc-hello-kubernetes:latest-alpine
docker run -p 8080:8080 --rm myc-hello-kubernetes:latest-debian-slim
docker run -p 8080:8080 --rm myc-hello-kubernetes:latest-distroless
docker run -p 8080:8080 --rm myc-hello-kubernetes:latest-ubi

curl http://localhost:8080/
```

Debug container:

```bash
docker run -it --rm --entrypoint=/bin/sh --user root -p 8080:8080 myc-hello-kubernetes:latest
```

Run in Kubernetes:

```bash
kubectl run myc-hello-kubernetes --image=quay.io/petr_ruzicka/myc-hello-kubernetes:latest
```

## Cosign - verify signatures

Example repositories:

- <https://quay.io/repository/cilium/cilium?tab=tags>
- <https://quay.io/repository/metallb/controller?tab=tags>
- <https://quay.io/repository/kubescape/kubescape?tab=tags>

```bash
IMAGE="registry.k8s.io/kube-apiserver-amd64:v1.24.0"

skopeo inspect --raw "docker://${IMAGE}"
COSIGN_EXPERIMENTAL=1 cosign verify "${IMAGE}" | jq
COSIGN_EXPERIMENTAL=1 cosign verify "${IMAGE}" | jq -r '.[].optional| .Issuer + "-" + .Subject'

rekor-cli get --uuid 68a53d0e75463d805dc9437dda5815171502475dd704459a5ce3078edba96226 --format json | jq -r .Attestation | base64 --decode | jq
curl -s https://rekor.sigstore.dev/api/v1/log/entries/68a53d0e75463d805dc9437dda5815171502475dd704459a5ce3078edba96226 | jq
rekor-cli search --email toddysm_dev1@outlook.com
```

## Possible build examples

- Build container images with tag latest form main branch

  ```bash
  gh workflow run container-build.yml -f container_registry_push=true -f container_image_expires_after=30 -f container_image_skip_vulnerability_checks=true
  ```

- Tag "main" branch

  ```bash
  TAG="8.0.10"

  git tag "v${TAG}-beta.0" && git push origin "v${TAG}-beta.0"
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  # In case some containers has security vulnerabilities they will not be pushed to Container Registry by default
  # If you want to do a force push please use something like:
  gh workflow run container-build.yml --ref="v${TAG}-beta.0" -f container_registry_push=true -f container_image_expires_after=365 -f container_image_skip_vulnerability_checks=true

  git tag "v${TAG}-beta.1" && git push origin "v${TAG}-beta.1"
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  gh workflow run container-build.yml --ref="v${TAG}-beta.1" -f container_registry_push=true -f container_image_expires_after=365 -f container_image_skip_vulnerability_checks=true

  git tag "v${TAG}-rc.0" && git push origin "v${TAG}-rc.0"
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  gh workflow run container-build.yml --ref="v${TAG}-rc.0" -f container_registry_push=true -f container_image_expires_after=365 -f container_image_skip_vulnerability_checks=true

  git tag "v${TAG}-rc.1" && git push origin "v${TAG}-rc.1"
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  gh workflow run container-build.yml --ref="v${TAG}-rc.1" -f container_registry_push=true -f container_image_expires_after=365 -f container_image_skip_vulnerability_checks=true

  git tag "v${TAG}" && git push origin "v${TAG}"
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  gh workflow run container-build.yml --ref="v${TAG}" -f container_registry_push=true -f container_image_expires_after=365 -f container_image_skip_vulnerability_checks=true
  ```

- Run build process form "main" branch - expires in 300 days

  ```bash
  gh workflow run container-build.yml -f container_registry_push=true -f container_image_expires_after=300 -f container_image_skip_vulnerability_checks=true
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  ```

- Run build process form "fix" branch - expires in 30 days (manual workflow
  execution is needed)

  ```bash
  git checkout -b fix && git push
  gh workflow run container-build.yml --ref=fix -f container_registry_push=true -f container_image_expires_after=30 -f container_image_skip_vulnerability_checks=true
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  git checkout main && git branch -d fix && git push origin -d fix
  ```

- Run build process form "fix2" branch form PR - expires in 30 days
  (manual workflow execution is needed)

  ```bash
  git checkout -b fix2
  date > date
  git add date && git commit -m "date" && git push
  PR_URL=$(gh pr create --fill)
  gh workflow run container-build.yml --ref=fix2 -f container_registry_push=true -f container_image_expires_after=30 -f container_image_skip_vulnerability_checks=true
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  date > date
  git add date && git commit -m "date2" && git push
  gh workflow run container-build.yml --ref=fix2  -f container_registry_push=true -f container_image_expires_after=30 -f container_image_skip_vulnerability_checks=true
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  gh pr close --delete-branch "${PR_URL}"
  ```

- Scheduled build - expires in 300 days
