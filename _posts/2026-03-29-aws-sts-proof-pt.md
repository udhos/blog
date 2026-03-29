---
title: "Usando AWS STS para provar identidade"
date: 2026-03-29
---

[English Version](/blog/2026/03/29/aws-sts-proof-en.html)

![Identidade Gopher](/blog/docs/assets/gopher_identity.png)

# Usando AWS STS para Provar Identidade

A tarefa em mãos é: a aplicação A (o `cliente`) precisa se autenticar para a aplicação B (o `servidor`) sem compartilhar quaisquer credenciais.

Não queremos compartilhar credenciais por razões bem conhecidas. Por um lado, aumentaria o risco de vazamento de credenciais. Além disso, evitamos confiar tanto no servidor.

Como o cliente pode provar sua identidade para o servidor sem compartilhar credenciais?

Bem, poderíamos, é claro, criar mecanismos de autenticação personalizados. Uma solução personalizada certamente pode funcionar, mas pode ser um exagero para tratar muitos casos simples.

Se já estamos usando AWS, podemos meio que abusar do AWS STS para produzir alguma prova de identidade. A ideia é muito direta:

1 - Tanto o servidor quanto o cliente estão cientes de que o servidor só está disposto a atender solicitações para uma identidade específica do AWS IAM, como uma função IAM.

2 - O cliente pré-assina uma solicitação GetCallerIdentity usando suas próprias credenciais (para a função IAM acordada). Em seguida, ele envia a solicitação pré-assinada para o servidor.

3 - O servidor apenas invoca a solicitação pré-assinada e verifica a resposta. Se a resposta identifica a função IAM esperada, então o cliente está autenticado.

# A pré-assinatura

Como é o processo de pré-assinatura?

A função abaixo destaca o núcleo do processo de pré-assinatura.

```golang
// PresignGetCallerIdentity creates a presigned STS GetCallerIdentity request and returns the parameters.
func PresignGetCallerIdentity(awsConfig aws.Config) (*v4.PresignedHTTPRequest, error) {
	//
	// sts client
	//

	clientSts := sts.NewFromConfig(awsConfig)

	// create a presigned STS GetCallerIdentity request and use it
	// as the proof request target (URL + signed headers)
	presignClient := sts.NewPresignClient(clientSts)

	presigned, errPresign := presignClient.PresignGetCallerIdentity(context.TODO(),
		&sts.GetCallerIdentityInput{})
	if errPresign != nil {
		return nil, fmt.Errorf("presign error: %v", errPresign)
	}

	return presigned, nil
}
```

# A verificação

Como é o processo de verificação?

A função abaixo ilustra o núcleo do processo de verificação.

O ponto principal a notar é que a verificação consiste em enviar a solicitação pré-assinada diretamente para a AWS usando um cliente HTTP padrão, sem envolver o SDK da AWS. O servidor só precisa verificar a resposta da AWS para autenticar a identidade do cliente.

Note que o servidor não precisa saber nada sobre as credenciais do cliente. Ele nem precisa possuir credenciais AWS. O servidor só precisa saber qual função IAM espera que o cliente esteja usando, e ele precisa de acesso online à API da AWS para verificar se a resposta da AWS identifica a função exigida.

```golang
// VerifyPresignedGetCallerIdentity forwards the presigned request to AWS and returns the response.
func VerifyPresignedGetCallerIdentity(ctx context.Context, client *http.Client,
	presigned *v4.PresignedHTTPRequest) (VerifyResponse, int, error) {

	//
	// Forward the presigned request to AWS using a plain HTTP client
	//

	var resp VerifyResponse

	// validate presigned request looks like STS GetCallerIdentity
	if presigned.Method == "" {
		return resp, http.StatusBadRequest, fmt.Errorf("missing method in presigned request")
	}
	if presigned.URL == "" {
		return resp, http.StatusBadRequest, fmt.Errorf("missing url in presigned request")
	}

	if presigned.Method != "GET" {
		return resp, http.StatusBadRequest, fmt.Errorf("presigned request must be GET: method=%q", presigned.Method)
	}

	u, errParse := url.Parse(presigned.URL)
	if errParse != nil {
		return resp, http.StatusBadRequest, fmt.Errorf("invalid url in presigned request: url=%q: %v",
			presigned.URL, errParse)
	}

	// require Query Action=GetCallerIdentity
	if action := u.Query().Get("Action"); action != "GetCallerIdentity" {
		return resp, http.StatusBadRequest,
			fmt.Errorf("presigned request Action is not GetCallerIdentity: action=%q", action)
	}

	// require signature: either Authorization header or X-Amz-Signature in query or headers
	hasAuth := false
	if _, ok := presigned.SignedHeader["Authorization"]; ok {
		hasAuth = true
	}
	if u.Query().Get("X-Amz-Signature") != "" {
		hasAuth = true
	}
	if _, ok := presigned.SignedHeader["X-Amz-Signature"]; ok {
		hasAuth = true
	}
	if !hasAuth {
		return resp, http.StatusBadRequest, fmt.Errorf("presigned request missing signature")
	}

	reqToAws, errReq := http.NewRequestWithContext(ctx, presigned.Method, presigned.URL, nil)
	if errReq != nil {
		return resp, http.StatusBadRequest, fmt.Errorf("error creating request to AWS: %v", errReq)
	}
	for k, vv := range presigned.SignedHeader {
		for _, v := range vv {
			reqToAws.Header.Add(k, v)
		}
	}

	respAws, errDo := client.Do(reqToAws)
	if errDo != nil {
		return resp, http.StatusBadGateway, fmt.Errorf("error forwarding request to AWS: %v", errDo)
	}
	defer respAws.Body.Close()

	respData, errRead := io.ReadAll(respAws.Body)
	if errRead != nil {
		return resp, http.StatusBadGateway, fmt.Errorf("error reading response from AWS: %v", errRead)
	}

	if respAws.StatusCode != 200 {
		return resp, http.StatusBadGateway, fmt.Errorf("bad status=%d body:%s", respAws.StatusCode, string(respData))
	}

	// Parse STS GetCallerIdentity XML response

	if err := xml.Unmarshal(respData, &resp); err != nil {
		log.Printf("xml unmarshal error: %v", err)
		// return raw AWS body for debugging
		return resp, http.StatusBadGateway, fmt.Errorf("error parsing AWS response: %v", err)
	}

	return resp, http.StatusOK, nil
}
```

# Um exemplo completo

Você pode clonar o projeto abaixo com exemplos de código totalmente funcionais para cliente e servidor.

[aws-sts-proof](https://github.com/udhos/aws-sts-proof) is a lightweight implementation of IAM-based authentication using signed GetCallerIdentity requests.
