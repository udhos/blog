---
title: "Imagem de Contêiner com Executável Único"
date: 2025-11-19
---

[Version in English](/blog/2025/11/19/single-file-container-image-en.html)

![Gopher Container](/blog/docs/assets/gopher_docker.png)

# Introdução

A linguagem Go permite que facilmente compilemos nossa aplicação em um único binário estático, sem dependências externas.

Além da conveniência de ser um único arquivo portátil, o binário estático geralmente é relativamente pequeno, especialmente para aplicações simples. Tamanhos típicos são da ordem de dezenas de megabytes.

Esse arranjo compacto abre a possibilidade de criar imagens de contêiner Docker extremamente pequenas, contendo apenas o binário estático e nada mais. As imagens pequenas contribuem com diversas vantagens:

- Downloads mais rápidos, reduzindo tempo de implantação/inicialização, tempo de escalonamento e custos de rede.
- Menor uso de espaço em disco, reduzindo custos de armazenamento.
- Superfície de ataque reduzida, aumentando a segurança.

A seguir, ilustraremos como compilar uma aplicação Go em um único binário estático e criar uma imagem de contêiner Docker mínima contendo apenas esse binário.

# Compilando para um Executável Único

Considere o seguinte comando de compilação.

```bash
$ CGO_ENABLED=0 go install -ldflags='-s -w' -trimpath ./app
```

Podemos verificar que o binário gerado é um executável estático usando o comando `file`. Note o trech `statically linked` na saída.

```bash
$ file `which app`

/home/everton/go/bin/app: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, BuildID[sha1]=c8a2bee2f51925e2a3b1f2711d116795555588c2, stripped
```

O arquivo binário estático executável gerado possui apenas 6 MB, pois se trata de uma aplicação simples.

```bash
$ ls -al `which app`
-rwxrwxr-x 1 everton everton 6103224 nov 19 21:55 /home/everton/go/bin/app
```

É importante lembrar que este único arquivo possui tudo o que é necessário para a execução da aplicação.

Podemos criar uma imagem de contêiner Docker que contenha apenas este único arquivo binário.

# Criando uma Imagem de Contêiner com um Único Arquivo

O Dockerfile abaixo mostra como fazer usando build em dois estágios. No primeiro estágio, usamos a imagem oficial do Go para compilar o binário estático. No segundo estágio, usamos a imagem `scratch`, que é uma imagem vazia, para criar uma imagem de contêiner mínima contendo apenas o binário compilado.

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

Para construir a image da receita acima, usamos o comando `docker build`:

```bash
docker build -t udhos/web-scratch:latest .
```
Após a construção, podemos verificar o tamanho da imagem usando o comando `docker images`:

```bash
$ docker images udhos/web-scratch:latest
REPOSITORY          TAG       IMAGE ID       CREATED          SIZE
udhos/web-scratch   latest    b7a2982cab1f   33 minutes ago   6.1MB
```

Mas será que a imagem funciona corretamente? Vamos testar executando um contêiner a partir da imagem criada:

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

Sim! Basta acessar `http://localhost:8080` no navegador para ver a aplicação em funcionamento.

![Browser Screenshot](/blog/docs/assets/gopher_docker_screenshot.png)

# Conclusão

A facilidade para criação de executáveis estáticos em Go, combinada com a imagem base `scratch` do Docker, permite criar imagens de contêiner extremamente pequenas e eficientes.

# Referências

- [https://github.com/udhos/goscratchello](https://github.com/udhos/goscratchello) - Projeto de exemplo completo com Dockerfile.

- [https://hub.docker.com/r/udhos/web-scratch](https://hub.docker.com/r/udhos/web-scratch) - Imagem publicada no Docker Hub.
