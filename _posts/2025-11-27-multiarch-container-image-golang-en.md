---
title: "Generating Multiarch Container Image with Go"
date: 2025-11-27
---

[Versão em Português](/blog/2025/11/27/multiarch-container-image-golang-pt.html)

![Gopher Multiarch](/blog/docs/assets/gopher_multiarch.png)

# Multiarch Container Image

For economic reasons, we often need to run our application on different CPU architectures.

A multi-architecture container image can serve workloads running on different architectures. The container runtime fetches the correct image according to the architecture of the machine where it is running.

The multiarch image is especially useful when we are migrating our application between machines with different architectures: the deployment recipe can use exactly the same image, regardless of the host machine's architecture.

Languages that do not have good support for cross-compilation may present some challenges when creating multi-arch images. Fortunately, Go has excellent support for cross-compilation, which greatly simplifies generating multi-architecture images.

# Generating Multiarch Container Image with Go

Step 1 of 3: Parameterize the Dockerfile to receive the `GOARCH` variable.

```Dockerfile
ARG GOARCH
RUN echo "Building for GOARCH=$GOARCH"
RUN go build -o /app ./app
```

Step 2 of 3: Generate an image for each desired architecture.

Go’s cross-compiling support will compile for the architecture defined in `GOARCH`.

```bash
app=docker.io/my-user/my-app:1.0.0

docker build --no-cache \
   --push \
   --build-arg GOARCH=amd64 \
   -t ${app}-amd64 \
   -f docker/Dockerfile .

docker build --no-cache \
   --push \
   --build-arg GOARCH=arm64 \
   -t ${app}-arm64 \
   -f docker/Dockerfile .
```

Step 3 of 3: Create the multi-architecture manifest for the images.

```bash
docker manifest create ${app} ${app}-amd64 ${app}-arm64
docker manifest annotate --arch amd64 ${app} ${app}-amd64
docker manifest annotate --arch arm64 ${app} ${app}-arm64
docker manifest push ${app}

# If you need to update the "latest" tag:

latest=docker.io/my-user/my-app:latest
docker manifest create ${latest} ${app}-amd64 ${app}-arm64
docker manifest annotate --arch amd64 ${latest} ${app}-amd64
docker manifest annotate --arch arm64 ${latest} ${app}-arm64
docker manifest push ${latest}
```

# Conclusion

Multi-architecture container images greatly facilitate managing workloads across machines with different architectures.

Go's first-class support for cross-compilation to multiple CPU architectures removes much of the friction involved in creating multi-architecture images.

# References

- Example of multiarch Dockerfile for Go: [Dockerfile](https://github.com/udhos/apiping/blob/main/docker/Dockerfile)

- Example of multiarch build script for Go: [build-multi-arch.sh](https://github.com/udhos/apiping/blob/main/docker/build-multi-arch.sh)

- [Migrating from x86 to AWS Graviton on Amazon EKS using Karpenter](https://aws.amazon.com/blogs/containers/migrating-from-x86-to-aws-graviton-on-amazon-eks-using-karpenter/)
