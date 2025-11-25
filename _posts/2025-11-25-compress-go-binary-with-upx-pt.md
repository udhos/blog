---
title: "Comprimindo Binário Go com UPX"
date: 2025-11-25
---

[Version in English](/blog/2025/11/25/compress-go-binary-with-upx-en.html)

![Gopher Container](/blog/docs/assets/gopher_upx.png)

# UPX

O [upx](https://upx.github.io/) (Ultimate Packer for eXecutables) é uma ferramenta de compressão de executáveis que pode reduzir significativamente o tamanho de binários compilados, incluindo aqueles gerados pela linguagem Go.

```bash
cd /tmp
wget https://github.com/upx/upx/releases/download/v5.0.2/upx-5.0.2-amd64_linux.tar.xz
tar xf upx-5.0.2-amd64_linux.tar.xz
/tmp/upx-5.0.2-amd64_linux/upx -9 caminho-para-o-binario
```

# Resultados com binário Go

Quando compilamos a aplicação Go sem fazer strip, obtemos um binário com 8914107 bytes:

```bash
go install ./app
```

Aplicando o upx ao binário sem strip, o tamanho reduz para 4850156 bytes:

```bash
/tmp/upx-5.0.2-amd64_linux/upx -9 ~/go/bin/app
```

Quando compilamos a aplicação Go com strip, o tamanho do binário fica em 6177060 bytes:

```bash
go install -ldflags='-s -w' ./app
```

Aplicando o upx ao binário com strip, o tamanho reduz para 2263460 bytes:

```bash
/tmp/upx-5.0.2-amd64_linux/upx -9 ~/go/bin/app
```

Vide resumo na tabela abaixo:

| Configuração              | Tamanho (bytes) | Redução (%) |
|---------------------------|-----------------|-------------|
| Sem strip                 | 8914107         | 0%          |
| Com strip                 | 6177060         | 30.7%       |
| Sem strip, com upx        | 4850156         | 45.6%       |
| Com strip, com upx        | 2263460         | 74.6%       |

# Resultados com imagem no Docker Hub

No Docker Hub, a imagem com strip (sem upx), cujo executável aparece acima com 6177060 bytes, ficou com 2.49 MB.

Aplicando o upx ao binário com strip, o tamanho da imagem Docker foi reduzido para 2.12 MB.

A redução do tamanho final da imagem Docker foi de 14.8%.

# Conclusão

O upx produziu boa redução no tamanho binário do Go: ~ 45% sem strip e ~ 63% em relação ao binário com strip. 

Na image no Docker Hub, para binário com strip, a redução do upx foi menor (~ 14%), pois o registry já fornece compressão eficiente.
