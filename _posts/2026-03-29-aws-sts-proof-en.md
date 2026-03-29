---
title: "Using AWS STS to prove identity"
date: 2026-03-29
---

[Versão em Português](/blog/2026/03/29/aws-sts-proof-pt.html)

![Gopher Identity](/blog/docs/assets/gopher_identity.png)

# Using AWS STS to Prove Identity

Task at hands is: application A (the `client`) needs to authenticate itself to application B (the `server`) without sharing any credentials.

We don't want to share credentials for well-known reasons. For one, it increases the risk of credential leakage. Also we avoid trusting the server to take care of our credentials.

How can the client prove its identity to the server without sharing credentials?

Well we could of course devise custom authentication mechanisms. A crafted solution certainly can work, but it might be an overkill in many simple cases.

If we are already using AWS, we can kind of abuse AWS STS to provide some proof of identity. The idea is straightforward:

1 - Both server and client are aware the server is only willing to provide requests for a specific AWS IAM identity, like an IAM role.

2 - The client presigns a GetCallerIdentity request using its own credentials (for the IAM role agreed upon). Then it sends the presigned request to the server.

3 - The server just invokes the presigned request and checks the response. If the response identifies the expected IAM role, then the client is authenticated.

# The presigning

What the presigning process looks like?

The function below factors out the core of the presigning process.

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

# The verification

What about the verification process?

The function below illustrates the core of the verification process.

Main point to note is that the verification consists of sending the presigned request directly to AWS using a plain HTTP client, without any AWS SDK involved. The server just needs to check the response from AWS to verify the client's identity.

Take notice that the server doesn't need to know anything about the client's credentials. It does not have to own AWS credentials. The server only needs to know which IAM role it expects the client to be using, and it needs live access to AWS API in order to verify that the response from AWS identifies the required role.

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

# A full example

You can clone the project below with full working code examples for both client and server.

[aws-sts-proof](https://github.com/udhos/aws-sts-proof) is a lightweight implementation of IAM-based authentication using signed GetCallerIdentity requests.
