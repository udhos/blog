---
title: "Go with Single Executable File Container Image"
date: 2025-11-19
---

[Versão em Português](/blog/2025/11/19/single-file-container-image-pt.html)

![Gopher Container](/blog/docs/assets/golang_docker.png)

# Introduction

The Go language makes it easy to compile our application into a single static binary, free from external dependencies.

In addition to the convenience of being a single portable file, the static binary is comparatively small, especially for simple applications. Typical sizes are on the order of tens of megabytes.

This compactness enables the creation of small Docker container images containing only the static binary and nothing else. Small images offer several advantages:

- Faster downloads, reducing deployment/startup time, scaling time, and network costs.
- Lower disk space usage, reducing storage costs.
- Reduced attack surface, increasing security.

Next, we will illustrate how to compile a Go application into a single static binary and create a minimal Docker container image containing only that binary.

# Building a Single Executable

Consider the following build command.

```bash
$ CGO_ENABLED=0 go install -ldflags='-s -w' -trimpath ./app
```

We can verify that the generated binary is a static executable using the `file` command. Note the `statically linked` part in the output.

```bash
$ file `which app`

/home/everton/go/bin/app: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, BuildID[sha1]=c8a2bee2f51925e2a3b1f2711d116795555588c2, stripped
```

The size of the generated static executable binary file is only 6 MB.

```bash
$ ls -al `which app`
-rwxrwxr-x 1 everton everton 6103224 nov 19 21:55 /home/everton/go/bin/app
```

It is important to remember that this single file contains everything needed to run the application.

We can create a Docker container image that contains only this single binary file.

# Creating a Single File Container Image

The Dockerfile below shows a two-stage image build. In the first stage, we use the official Go image to compile the static binary. In the second stage, we use the `scratch` image, which is an empty image, to create a minimal container image containing only the compiled binary.

```Dockerfile
# STEP 1 build executable binary

FROM golang:1.25.4-alpine AS builder

RUN adduser -D -g '' appuser
COPY app/* /build/app/
COPY go.mod /build/
WORKDIR /build
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags='-s -w' -trimpath -o /app ./app

# STEP 2 build a small image

FROM scratch

COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /app /app
USER appuser
ENTRYPOINT ["/app"]
```

To build the image from the recipe above, we use the `docker build` command:

```bash
docker build -t udhos/web-scratch:latest .
```

After the build, we can check the image size using the `docker images` command:

```bash
$ docker images udhos/web-scratch:latest
REPOSITORY          TAG       IMAGE ID       CREATED          SIZE
udhos/web-scratch   latest    b7a2982cab1f   33 minutes ago   6.1MB
```

6.1 MB!

But does the image work correctly? Let's run a container from the created image:

```bash
$ docker run --rm -p 8080:8080 -it udhos/web-scratch:latest
2025/11/20 01:14:42 version=0.8.2 runtime=go1.25.4 pid=1 host=ab8b011a7ada GOMAXPROCS=16 OS=linux ARCH=amd64
2025/11/20 01:14:42 GWH_BANNER='' PORT=''
2025/11/20 01:14:42 user: appuser (uid: 1000)
2025/11/20 01:14:42 banner: banner default
2025/11/20 01:14:42 keepalive: true
2025/11/20 01:14:42 burnCpu=false
2025/11/20 01:14:42 quotaTime=0s
2025/11/20 01:14:42 requests quota=0 (0=unlimited) exitOnQuota=false quotaStatus=500
2025/11/20 01:14:42 TLS key file not found: key.pem - disabling TLS
2025/11/20 01:14:42 TLS cert file not found: cert.pem - disabling TLS
2025/11/20 01:14:42 mapping www path /www/ to directory /
2025/11/20 01:14:42 using TCP ports HTTP=:8080 HTTPS=:8443 TLS=false
2025/11/20 01:14:42 serving HTTP on TCP :8080
```

It works! Just access `http://localhost:8080` in your browser to see the application running.

![Browser Screenshot](/blog/docs/assets/golang_docker_screenshot.png)

# Conclusion

The ease of creating static executables in Go, combined with Docker's `scratch` base image, allows for the creation of extremely small and efficient container images.

# References

- [https://github.com/udhos/goscratchello](https://github.com/udhos/goscratchello) - Full example project with Dockerfile.

- [https://hub.docker.com/r/udhos/web-scratch](https://hub.docker.com/r/udhos/web-scratch) - Image published on Docker Hub.
