# Container build repository

Requirements:

- Container scan failed (how to progress even when there are CRITICAL
  vulnerabilities in the container)
- I want to build test container (with specific tag like
  `myc-hello-kubernetes:5.1.2-test1` with expiration date)
- I want to build test container for specific PR with expiration date
- I want to build only `latest` tag manually ?
- Newly released images must create/update following tags in container
  repository (example):
  - `1`
  - `1.2`
  - `1.2.0`
  - `latest`
- Container image should be scanned for "Critical" vulnerabilities
- Image should be signed ([cosign](https://github.com/sigstore/cosign))
- SBOM should be generated
