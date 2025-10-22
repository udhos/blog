---
title: "Who Canceled My Golang HTTP Context?"
date: 2025-10-21
---

[Versão em Português](/blog/2025/10/21/context-canceled-pt.html)

![Squad](/blog/docs/assets/gopher_stop_sign.png)

# Package context

The [package context documentation](https://pkg.go.dev/context) introduces the context concept thus:

> Package context defines the Context type, which carries deadlines, cancellation signals, and other request-scoped values across API boundaries and between processes.
> 
> Incoming requests to a server should create a Context, and outgoing calls to servers should accept a Context.

Therefore in http handlers we retrieve the context from http requests and then we propagate it into other pieces.

Since we check properly for errors, and we also log unexpected errors, we are likely to soon find unclear `context canceled` errors popping up in our production logs.

```
The database query returned this error: context canceled
```

Is that even an error?

# Who Did Canceled My Golang HTTP Context?

By double-checking the application code, we feel assured that it didn't cancel the context.

Who did?

One usual suspect is the library that created the context for us, if any.

For instance, the HTTP server package from the standard library is known for cancelling requests whenever it detects the client connection has been broken. Since the http caller is gone, there is little point in finishing the request.

But now the `canceled context` condition is going to be reported as error by any service for whom we forwarded the context. It is not an actual error, but it is hard to tell it from a real error because it does not provide the cause.

# How to Prevent This Ambiguity?

From Go 1.20 onwards, the `context` package provides a new function to create cancellable contexts with explicit causes:

```go
ctx, cancelWithCause := context.WithCancelCause(context.Background())
// ...
cancelWithCause(fmt.Errorf("client connection has been closed: %w", err))
```

Then we should always be polite and provide the explicit cause when cancelling a context. The cause of cancellation will likely be logged in our error checking code, making it clear why the context was canceled.

What about existing libraries that cancel contexts without providing a cause?

Yes, they are a pain. While we wait for them to be updated, we must be aware that unclear `context canceled` messages may appear in our logs and might not signal actual errors!

# Key Takeaways

1. Whenever you need to cancel a context, be polite and provide the clear explicit cause.

```go
ctx, cancelWithCause := context.WithCancelCause(context.Background())
// ...
cancelWithCause(fmt.Errorf("client connection has been closed: %w", err))
```

2. `context.WithCancelCause` has been added in Go 1.20. Libraries predating that version are likely to fail in providing a cause when cancelling their contexts. Hence be prepared to deal with unexpected `context canceled` errors from those libraries.

3. The http server from standard library is known for cancelling contexts with a cause. There is hope for the feature (see the issue below).

# References

[proposal: net/http: add context cancelation reason for server handlers](https://github.com/golang/go/issues/64465)
