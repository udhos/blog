---
title: "Gerando Imagem de Contêiner Multiarquitetura com Go"
date: 2025-11-27
---

[Version in English](/blog/2025/11/27/multiarch-container-image-golang-en.html)

![Gopher Multiarch](/blog/docs/assets/gopher_multiarch.png)

# Imagem Multiarquitetura

Por razões econômicas, frequentemente precisamos rodar nossa aplicação em diferentes arquiteturas de CPU.

Uma imagem de contêiner multiarquitetura consegue atender cargas de trabalho executando em diferentes arquiteturas. O contêiner runtime obterá a imagem correta segundo a arquitetura da máquina onde está rodando.

A imagem multiarquitetura é especialmente útil quando estamos migrando nossa aplicação entre máquinas de diferentes arquiteturas: a receita de implantação pode utilizar exatamente a mesma imagem, independentemente da arquitetura da máquina hospedeira.

Linguagens que não possuem bom suporte para cross-compilation podem apresentar alguns desafios para criar imagens multiarch. Felizmente, o Go possui excelente suporte para cross-compilation, o que facilita imensamente a geração de imagem multiarquitetura.

# Gerando Imagem Multiarquitetura com Go

Passo 1 de 3: Parametrizar o Dockerfile para receber a variável `GOARCH`.

```Dockerfile
ARG GOARCH
RUN echo "Building for GOARCH=$GOARCH"
RUN go build -o /app ./app
```

Passo 2 de 3: Gerar uma imagem para cada arquitetura desejada.

O suporte a cross-compiling do Go compilará para a arquitetura definida na `GOARCH`.

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

Passo 3 de 3: Criar o manifesto multiarquitetura para as imagens.

```bash
docker manifest create ${app} ${app}-amd64 ${app}-arm64
docker manifest annotate --arch amd64 ${app} ${app}-amd64
docker manifest annotate --arch arm64 ${app} ${app}-arm64
docker manifest push ${app}

# Se precisar atualizar a tag "latest":

latest=docker.io/my-user/my-app:latest
docker manifest create ${latest} ${app}-amd64 ${app}-arm64
docker manifest annotate --arch amd64 ${latest} ${app}-amd64
docker manifest annotate --arch arm64 ${latest} ${app}-arm64
docker manifest push ${latest}
```

# Conclusão

Imagens de contêiner multiarquitetura facilitam muito o manejo de cargas de trabalho entre máquinas de diferentes arquiteturas.

O suporte de primeira classe da linguagem Go para cross-compilation para várias arquiteturas de CPU remove boa parte da fricção envolvida na criação de imagens multiarquitetura.

# Referências

- Exemplo de Dockerfile multiarch para Go: [Dockerfile](https://github.com/udhos/apiping/blob/main/docker/Dockerfile)

- Exemplo de script de build multiarch para Go: [build-multi-arch.sh](https://github.com/udhos/apiping/blob/main/docker/build-multi-arch.sh)

- [Migrating from x86 to AWS Graviton on Amazon EKS using Karpenter](https://aws.amazon.com/blogs/containers/migrating-from-x86-to-aws-graviton-on-amazon-eks-using-karpenter/)
