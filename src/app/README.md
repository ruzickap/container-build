# Hello Kubernetes demo app

The source code was taken from: <https://github.com/paulbouwer/hello-kubernetes>

## Notes

Build commands:

```bash
docker build -f ./Dockerfile -t hello-kubernetes .
docker build -f ./Dockerfile -t hello-kubernetes:node-18.11.0-alpine3.16 -f Dockerfile-node-18.11.0-alpine3.16 .
docker build -f ./Dockerfile -t hello-kubernetes:node-18.11.0-bullseye-slim -f Dockerfile-node-18.11.0-bullseye-slim .
docker build -f ./Dockerfile -t hello-kubernetes:ubi9-nodejs-16-minimal -f Dockerfile-ubi9-nodejs-16-minimal .
```

Run it:

```bash
docker run -p 8080:8080 --rm hello-kubernetes
docker run -p 8080:8080 --rm hello-kubernetes:node-18.11.0-alpine3.16
docker run -p 8080:8080 --rm hello-kubernetes:node-18.11.0-bullseye-slim
docker run -p 8080:8080 --rm hello-kubernetes:ubi9-nodejs-16-minimal
```
