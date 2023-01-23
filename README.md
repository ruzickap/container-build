# Container build process

[![container-build](https://github.com/ruzickap/container-build/actions/workflows/container-build.yml/badge.svg)](https://github.com/ruzickap/container-build/actions/workflows/container-build.yml)

The source code was taken from: <https://github.com/paulbouwer/hello-kubernetes>

Repository with images: <https://quay.io/repository/petr_ruzicka/myc-hello-kubernetes?tab=tags>

## Requirements

### Generic

- Multiple Dockerfile files in one repository
- [not done] Multi-arch container images for each Dockerfile
- Image should be signed ([cosign](https://github.com/sigstore/cosign))
- SBOM should be generated
- Build process should never overwrite existing tags (1.2.3) in container
  repository (doesn't apply for "latest" or "br-")
- Container image should be scanned for "Critical" vulnerabilities before it
  is pushed to container registry
- There should be a way to push container to registry also with CRITICAL
  vulnerabilities (how to progress when there are CRITICAL vulnerabilities which
  can not be fixed "immediately")
  - Set `container_image_vulnerability_checks=false`

### Tag specific

- I want to build test container for specific PR / branch with expiration date
- I want to build only `latest` tag manually (form `main` branch)
- Newly released images must create/update following tags in container
  repository (example):
  - `1`
  - `1.2`
  - `1.2.1`
  - `latest`

## Local tests

Build commands:

```bash
docker build -f ./src/app/Dockerfile                     -t myc-hello-kubernetes:latest             src/app
docker build -f ./src/app/Dockerfile-node-18-alpine      -t myc-hello-kubernetes:latest-alpine      src/app
docker build -f ./src/app/Dockerfile-node-18-debian-slim -t myc-hello-kubernetes:latest-debian-slim src/app
docker build -f ./src/app/Dockerfile-nodejs18-distroless -t myc-hello-kubernetes:latest-distroless  src/app
docker build -f ./src/app/Dockerfile-nodejs-16-ubi       -t myc-hello-kubernetes:latest-ubi         src/app
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
kubectl run myc-hello-kubernetes --image=quay.io/petr_ruzicka/myc-hello-kubernetes
```

## Cosign - verify signatures / SBOMs

Example repositories:

- <https://quay.io/repository/cilium/cilium?tab=tags>
- <https://quay.io/repository/metallb/controller?tab=tags>
- <https://quay.io/repository/kubescape/kubescape?tab=tags>
- <https://quay.io/repository/costoolkit/elemental-operator?tab=tags&tag=latest>
- <https://hub.docker.com/r/bridgecrew/checkov/tags>

Notes:

- [Attest and verify artifacts](https://rewanthtammana.com/sigstore-the-easy-way/cosign/attest-and-verify-artifacts/)
- <https://github.com/marco-lancini/utils/blob/main/ci/github/docker-build-sign-sbom/reusable-docker.yml>

Test commands:

```bash
IMAGE="bridgecrew/checkov"
IMAGE="quay.io/petr_ruzicka/malware-cryptominer-container:1.4.4"
IMAGE="ghcr.io/external-secrets/external-secrets:v0.7.1"
export COSIGN_EXPERIMENTAL=1

cosign verify "${IMAGE}" | jq
cosign verify "${IMAGE}" | jq -r '.[].optional| .Issuer + " | " + .Subject + " | " + .githubWorkflowRef + " | https://rekor.tlog.dev/?logIndex=" + (.Bundle.Payload.logIndex|tostring)'
cosign triangulate "${IMAGE}"
cosign tree "${IMAGE}"
cosign verify-attestation --type cyclonedx "${IMAGE}"
cosign verify-attestation --type cyclonedx "${IMAGE}" | jq '.payload |= @base64d | .payload | fromjson'
trivy image --debug --sbom-sources rekor "${IMAGE}"
cosign verify-attestation --type cyclonedx "${IMAGE}" | jq -r '.payload' | base64 -d | jq -r '.predicate.Data' > /tmp/sbom.cdx.json
trivy sbom /tmp/sbom.cdx.json
cosign verify-attestation --type slsaprovenance "${IMAGE}"
```

Test commands with other images:

```bash
IMAGE="registry.k8s.io/kube-apiserver-amd64:v1.24.0"
IMAGE="quay.io/petr_ruzicka/malware-cryptominer-container:latest"
IMAGE="quay.io/cilium/cilium:v1.13.0-rc3"
IMAGE="quay.io/metallb/controller:latest"
IMAGE="quay.io/costoolkit/elemental-operator:v1.1.0"
IMAGE="bridgecrew/checkov"
export COSIGN_EXPERIMENTAL=1

skopeo inspect --raw "docker://${IMAGE}"
cosign verify "${IMAGE}" | jq
cosign verify "${IMAGE}" | jq -r '.[].optional| .Issuer + "-" + .Subject'
cosign triangulate "${IMAGE}"
cosign tree "${IMAGE}"

rekor-cli get --uuid 68a53d0e75463d805dc9437dda5815171502475dd704459a5ce3078edba96226 --format json | jq -r .Attestation | base64 --decode | jq
curl -s https://rekor.sigstore.dev/api/v1/log/entries/68a53d0e75463d805dc9437dda5815171502475dd704459a5ce3078edba96226 | jq

rekor-cli search --email petr.ruzicka@gmail.com
# https://rekor.tlog.dev/?email=petr.ruzicka@gmail.com

rekor-cli get --log-index 8757761

docker manifest inspect "${IMAGE}"
cosign download sbom ghcr.io/google/ko --platform=linux/amd64
cosign download sbom --platform=linux/amd64 cgr.dev/chainguard/node
cosign download sbom "${IMAGE}"

# Image Manifest
curl -s -H 'Accept: application/vnd.docker.distribution.manifest.v2+json' "https://quay.io/v2/jetstack/cert-manager-controller/manifests/v1.9.1" | jq
# Manifest List
curl -s -H 'Accept: application/vnd.docker.distribution.manifest.list.v2+json' "https://quay.io/v2/jetstack/cert-manager-controller/manifests/v1.9.1" | jq
curl -s -H 'Accept: application/vnd.oci.image.index.v1+json' "https://quay.io/v2/jetstack/cert-manager-controller/manifests/v1.9.1" | jq

# Verify SBOM attestation
cosign verify-attestation --type cyclonedx bridgecrew/checkov | jq '.payload |= @base64d | .payload | fromjson | select(.predicateType == "https://cyclonedx.org/schema") | .predicate.Data'
cosign verify-attestation --type slsaprovenance --key https://ftp.suse.com/pub/projects/security/keys/container–key.pem registry.suse.com/bci/golang@sha256:35bc38ce40811b587a56bcfa328ef077c0703732e3bbedf4dbdf47f612cca04b | jq
cosign verify-attestation --type slsaprovenance ghcr.io/thomasvitale/band-service@sha256:388e8d292b55a7934bdaf11277ea9f33c3533258de92eb4b12085717dbdbd875 | jq '.payload |= @base64d | .payload | fromjson'

docker manifest inspect quay.io/jetstack/cert-manager-controller:v1.9.1


slsa-verifier verify-image "ghcr.io/chgl/kube-powertools@sha256:b76dc742957ae883f3ed0c4fe89b54d5c9a9de69a8bc531f9ee12ec995dab10d" --source-uri github.com/chgl/kube-powertools

https://registry-ui.chainguard.app/?image=quay.io/petr_ruzicka/malware-cryptominer-container:1
https://registry-ui.chainguard.app/?image=cgr.dev/chainguard/nginx:latest
```

## Possible build examples

- Build container images with tag latest form main branch and ignore
  vulnerability scanners

  ```bash
  gh workflow run container-build.yml -f container_registry_push=true -f container_image_expires_after=30 -f container_image_skip_vulnerability_checks=true
  sleep 100
  ```

- Tag "main" branch

  ```bash
  TAG="8.0.49"

  git tag "v${TAG}-beta.0" && git push origin "v${TAG}-beta.0"
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  # In case some containers has security vulnerabilities they will not be pushed to Container Registry by default
  # If you want to do a force push please use something like:
  gh workflow run container-build.yml --ref="v${TAG}-beta.0" -f container_registry_push=true -f container_image_expires_after=365 -f container_image_skip_vulnerability_checks=true
  sleep 100

  git tag "v${TAG}-beta.1" && git push origin "v${TAG}-beta.1"
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  gh workflow run container-build.yml --ref="v${TAG}-beta.1" -f container_registry_push=true -f container_image_expires_after=365 -f container_image_skip_vulnerability_checks=true
  sleep 100

  git tag "v${TAG}-rc.0" && git push origin "v${TAG}-rc.0"
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  gh workflow run container-build.yml --ref="v${TAG}-rc.0" -f container_registry_push=true -f container_image_expires_after=365 -f container_image_skip_vulnerability_checks=true
  sleep 100

  git tag "v${TAG}-rc.1" && git push origin "v${TAG}-rc.1"
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  gh workflow run container-build.yml --ref="v${TAG}-rc.1" -f container_registry_push=true -f container_image_expires_after=365 -f container_image_skip_vulnerability_checks=true
  sleep 100

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
  sleep 100
  date > date
  git add date && git commit -m "date2" && git push
  gh workflow run container-build.yml --ref=fix2  -f container_registry_push=true -f container_image_expires_after=30 -f container_image_skip_vulnerability_checks=true
  sleep 10
  WORKLOAD_ID=$(gh run list --workflow=container-build.yml --limit 1 --json databaseId | jq -r '.[].databaseId')
  gh run watch "${WORKLOAD_ID}"
  gh pr close --delete-branch "${PR_URL}"
  ```

- Scheduled build - expires in 300 days
