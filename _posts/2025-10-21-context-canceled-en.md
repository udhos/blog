---
title: "Who Canceled My Golang HTTP Context?"
date: 2025-10-21
---

[Versão em Português](/blog/2025/10/21/context-canceled-pt.html)

![Stop](/blog/docs/assets/gopher_stop_sign.png)

# Package context

The [package context documentation](https://pkg.go.dev/context) introduces the context concept as follows:

> Package context defines the Context type, which carries deadlines, cancellation signals, and other request-scoped values across API boundaries and between processes.
> 
> Incoming requests to a server should create a Context, and outgoing calls to servers should accept a Context.

Therefore, in HTTP handlers we retrieve the context from HTTP requests and then propagate it into other components.

Since we properly check for errors and also log unexpected errors, we are likely to soon find unclear `context canceled` errors popping up in our production logs.

```
The database query returned this error: context canceled
```

Is that even an error?

# Who Canceled My Golang HTTP Context?

Double-checking the application code, we are reassured that it didn't cancel the context.

Who did?

One usual suspect is the library that created the context for us, if any.

For instance, the HTTP server package from the standard library is known for cancelling requests whenever it detects the client connection has been broken. Since the HTTP caller is gone, there is little point in finishing the request.

But now the `context canceled` condition is going to be reported as an error by any service to which we forwarded the context. It is not even an actual error, but it is hard to tell it from a real error because it does not provide the cause.

# How to Prevent This Ambiguity?

From Go 1.20 onwards, the `context` package provides a new function to create cancellable contexts with explicit causes:

```go
ctx, cancelWithCause := context.WithCancelCause(context.Background())
// ...
cancelWithCause(fmt.Errorf("client connection has been closed: %w", err))
```

We should take advantage of this improvement to be polite and provide an explicit cause when cancelling a context. The cause of cancellation will likely be logged in our error checking code, making it clear why the context was cancelled.

What about existing libraries that cancel contexts without providing a cause?

Yes, there is pain. While we wait for them to be updated, we must be aware that unclear `context canceled` messages may appear in our logs and might not signal actual errors!

# Key Takeaways

1\. Whenever you need to cancel a context, be polite and provide a clear, explicit cause.

```go
ctx, cancelWithCause := context.WithCancelCause(context.Background())
// ...
cancelWithCause(fmt.Errorf("client connection has been closed: %w", err))
```

2\. `context.WithCancelCause` was added in Go 1.20. Libraries predating that version are likely to fail to provide a cause when cancelling their contexts. Hence, be prepared to deal with unexpected `context canceled` errors from those libraries.

3\. The HTTP server from the standard library is known for cancelling contexts without providing the cause. But there is hope for the future (see the issue below).

# References

[proposal: net/http: add context cancellation reason for server handlers](https://github.com/golang/go/issues/64465)
