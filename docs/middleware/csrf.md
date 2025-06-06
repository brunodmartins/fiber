---
id: csrf
---

# CSRF

The CSRF middleware for [Fiber](https://github.com/gofiber/fiber) provides protection against [Cross-Site Request Forgery](https://en.wikipedia.org/wiki/Cross-site_request_forgery) (CSRF) attacks. Requests made using methods other than those defined as 'safe' by [RFC9110#section-9.2.1](https://datatracker.ietf.org/doc/html/rfc9110.html#section-9.2.1) (GET, HEAD, OPTIONS, and TRACE) are validated using tokens. If a potential attack is detected, the middleware will return a default 403 Forbidden error.

This middleware offers two [Token Validation Patterns](#token-validation-patterns): the [Double Submit Cookie Pattern (default)](#double-submit-cookie-pattern-default), and the [Synchronizer Token Pattern (with Session)](#synchronizer-token-pattern-with-session).

As a [Defense In Depth](#defense-in-depth) measure, this middleware performs [Referer Checking](#referer-checking) for HTTPS requests.

## How to use Fiber's CSRF Middleware

## Examples

Import the middleware package that is part of the Fiber web framework:

```go
import (
    "github.com/gofiber/fiber/v3"
    "github.com/gofiber/fiber/v3/middleware/csrf"
)
```

After initializing your Fiber app, you can use the following code to initialize the middleware:

```go
// Initialize default config
app.Use(csrf.New())

// Or extend your config for customization
app.Use(csrf.New(csrf.Config{
    KeyLookup:      "header:X-Csrf-Token",
    CookieName:     "csrf_",
    CookieSameSite: "Lax",
    IdleTimeout:    30 * time.Minute,
    KeyGenerator:   utils.UUIDv4,
    Extractor:      func(c fiber.Ctx) (string, error) { ... },
}))
```

:::info
KeyLookup will be ignored if Extractor is explicitly set.
:::

Getting the CSRF token in a handler:

```go
func handler(c fiber.Ctx) error {
    // Get CSRF token from the context
    token := csrf.TokenFromContext(c)
    if token == "" {
        return c.Status(fiber.StatusInternalServerError)
    }
    
    // Note: Make sure this matches the KeyLookup configured in your middleware
    // Example: If you configured csrf.Config{KeyLookup: "form:_csrf"}
    formKey := "_csrf"
    
    // Create a form with the CSRF token
    tmpl := fmt.Sprintf(`<form action="/post" method="POST">
        <input type="hidden" name="%s" value="%s">
        <input type="text" name="message">
        <input type="submit" value="Submit">
    </form>`, formKey, token)
    
    c.Set("Content-Type", "text/html")
    return c.SendString(tmpl)
}
```

## Recipes for Common Use Cases

There are two basic use cases for the CSRF middleware:

1. **Without Sessions**: This is the simplest way to use the middleware. It uses the Double Submit Cookie Pattern and does not require a user session.

    - See GoFiber recipe [CSRF](https://github.com/gofiber/recipes/tree/master/csrf) for an example of using the CSRF middleware without a user session.

2. **With Sessions**: This is generally considered more secure. It uses the Synchronizer Token Pattern and requires a user session, and the use of pre-session, which prevents login CSRF attacks.

    - See GoFiber recipe [CSRF with Session](https://github.com/gofiber/recipes/tree/master/csrf-with-session) for an example of using the CSRF middleware with a user session.

## Signatures

```go
func New(config ...Config) fiber.Handler
func TokenFromContext(c fiber.Ctx) string
func HandlerFromContext(c fiber.Ctx) *Handler

func (h *Handler) DeleteToken(c fiber.Ctx) error
```

## Config

| Property          | Type                               | Description                                                                                                                                                                                                                                                                                  | Default                      |
|:------------------|:-----------------------------------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:-----------------------------|
| Next              | `func(fiber.Ctx) bool`             | Next defines a function to skip this middleware when returned true.                                                                                                                                                                                                                          | `nil`                        |
| KeyLookup         | `string`                           | KeyLookup is a string in the form of "`<source>:<key>`" that is used to create an Extractor that extracts the token from the request. Possible values: "`header:<name>`", "`query:<name>`", "`param:<name>`", "`form:<name>`", "`cookie:<name>`". Ignored if an Extractor is explicitly set. | "header:X-CSRF-Token"        |
| CookieName        | `string`                           | Name of the csrf cookie. This cookie will store the csrf key.                                                                                                                                                                                                                                | "csrf_"                      |
| CookieDomain      | `string`                           | Domain of the CSRF cookie.                                                                                                                                                                                                                                                                   | ""                           |
| CookiePath        | `string`                           | Path of the CSRF cookie.                                                                                                                                                                                                                                                                     | ""                           |
| CookieSecure      | `bool`                             | Indicates if the CSRF cookie is secure.                                                                                                                                                                                                                                                      | false                        |
| CookieHTTPOnly    | `bool`                             | Indicates if the CSRF cookie is HTTP-only.                                                                                                                                                                                                                                                   | false                        |
| CookieSameSite    | `string`                           | Value of SameSite cookie.                                                                                                                                                                                                                                                                    | "Lax"                        |
| CookieSessionOnly | `bool`                             | Decides whether the cookie should last for only the browser session. (cookie expires on close).                                                                                                                                                                                              | false                        |
| IdleTimeout       | `time.Duration`                    | IdleTimeout is the duration of inactivity before the CSRF token will expire.                                                                                                                                                                                                                 | 30 * time.Minute             |
| KeyGenerator      | `func() string`                    | KeyGenerator creates a new CSRF token.                                                                                                                                                                                                                                                       | utils.UUID                   |
| ErrorHandler      | `fiber.ErrorHandler`               | ErrorHandler is executed when an error is returned from fiber.Handler.                                                                                                                                                                                                                       | DefaultErrorHandler          |
| Extractor         | `func(fiber.Ctx) (string, error)`  | Extractor returns the CSRF token. If set, this will be used in place of an Extractor based on KeyLookup.                                                                                                                                                                                     | Extractor based on KeyLookup |
| SingleUseToken    | `bool`                             | SingleUseToken indicates if the CSRF token be destroyed and a new one generated on each use. (See TokenLifecycle)                                                                                                                                                                            | false                        |
| Storage           | `fiber.Storage`                    | Store is used to store the state of the middleware.                                                                                                                                                                                                                                          | `nil`                        |
| Session           | `*session.Store`                   | Session is used to store the state of the middleware. Overrides Storage if set.                                                                                                                                                                                                              | `nil`                        |
| TrustedOrigins    | `[]string`                         | TrustedOrigins is a list of trusted origins for unsafe requests. This supports subdomain matching, so you can use a value like "https://*.example.com" to allow any subdomain of example.com to submit requests.                                                                             | `[]`                         |

### Default Config

```go
var ConfigDefault = Config{
    KeyLookup:         "header:" + HeaderName,
    CookieName:        "csrf_",
    CookieSameSite:    "Lax",
    IdleTimeout:       30 * time.Minute,
    KeyGenerator:      utils.UUIDv4,
    ErrorHandler:      defaultErrorHandler,
    Extractor:         FromHeader(HeaderName),
}
```

### Recommended Config (with session)

It's recommended to use this middleware with [fiber/middleware/session](https://docs.gofiber.io/api/middleware/session) to store the CSRF token within the session. This is generally more secure than the default configuration.

```go
var ConfigDefault = Config{
    KeyLookup:         "header:" + HeaderName,
    CookieName:        "__Host-csrf_",
    CookieSameSite:    "Lax",
    CookieSecure:       true,
    CookieSessionOnly: true,
    CookieHTTPOnly:    true,
    IdleTimeout:       30 * time.Minute,
    KeyGenerator:      utils.UUIDv4,
    ErrorHandler:      defaultErrorHandler,
    Extractor:         FromHeader(HeaderName),
    Session:           session.Store,
}
```

### Trusted Origins

The `TrustedOrigins` option is used to specify a list of trusted origins for unsafe requests. This is useful when you want to allow requests from other origins. This supports matching subdomains at any level. This means you can use a value like `"https://*.example.com"` to allow any subdomain of `example.com` to submit requests, including multiple subdomain levels such as `"https://sub.sub.example.com"`.

To ensure that the provided `TrustedOrigins` origins are correctly formatted, this middleware validates and normalizes them. It checks for valid schemes, i.e., HTTP or HTTPS, and it will automatically remove trailing slashes. If the provided origin is invalid, the middleware will panic.

#### Example with Explicit Origins

In the following example, the CSRF middleware will allow requests from `trusted.example.com`, in addition to the current host.

```go
app.Use(csrf.New(csrf.Config{
    TrustedOrigins: []string{"https://trusted.example.com"},
}))
```

#### Example with Subdomain Matching

In the following example, the CSRF middleware will allow requests from any subdomain of `example.com`, in addition to the current host.

```go
app.Use(csrf.New(csrf.Config{
    TrustedOrigins: []string{"https://*.example.com"},
}))
```

:::caution
When using `TrustedOrigins` with subdomain matching, make sure you control and trust all the subdomains, including all subdomain levels. If not, an attacker could create a subdomain under a trusted origin and use it to send harmful requests.
:::

## Constants

```go
const (
    HeaderName = "X-Csrf-Token"
)
```

## Sentinel Errors

The CSRF middleware utilizes a set of sentinel errors to handle various scenarios and communicate errors effectively. These can be used within a [custom error handler](#custom-error-handler) to handle errors returned by the middleware.

### Errors Returned to Error Handler

- `ErrTokenNotFound`: Indicates that the CSRF token was not found.
- `ErrTokenInvalid`: Indicates that the CSRF token is invalid.
- `ErrRefererNotFound`: Indicates that the referer was not supplied.
- `ErrRefererInvalid`: Indicates that the referer is invalid.
- `ErrRefererNoMatch`: Indicates that the referer does not match host and is not a trusted origin.
- `ErrOriginInvalid`: Indicates that the origin is invalid.
- `ErrOriginNoMatch`: Indicates that the origin does not match host and is not a trusted origin.

If you use the default error handler, the client will receive a 403 Forbidden error without any additional information.

## Custom Error Handler

You can use a custom error handler to handle errors returned by the CSRF middleware. The error handler is executed when an error is returned from the middleware. The error handler is passed the error returned from the middleware and the fiber.Ctx.

Example, returning a JSON response for API requests and rendering an error page for other requests:

```go
app.Use(csrf.New(csrf.Config{
    ErrorHandler: func(c fiber.Ctx, err error) error {
        accepts := c.Accepts("html", "json")
        path := c.Path()
        if accepts == "json" || strings.HasPrefix(path, "/api/") {
            return c.Status(fiber.StatusForbidden).JSON(fiber.Map{
                "error": "Forbidden",
            })
        }
        return c.Status(fiber.StatusForbidden).Render("error", fiber.Map{
            "Title": "Forbidden",
            "Status": fiber.StatusForbidden,
        }, "layouts/main")
    },
}))
```

## Custom Storage/Database

You can use any storage from our [storage](https://github.com/gofiber/storage/) package.

```go
storage := sqlite3.New() // From github.com/gofiber/storage/sqlite3
app.Use(csrf.New(csrf.Config{
    Storage: storage,
}))
```

## How It Works

### Token Generation

CSRF tokens are generated on 'safe' requests and when the existing token has expired or hasn't been set yet. If `SingleUseToken` is `true`, a new token is generated after each use.  Retrieve the CSRF token using `csrf.TokenFromContext(c)`.

### Security Considerations

This middleware is designed to protect against CSRF attacks but does not protect against other attack vectors, such as XSS. It should be used in combination with other security measures.

:::danger
Never use 'safe' methods to mutate data, for example, never use a GET request to modify a resource. This middleware will not protect against CSRF attacks on 'safe' methods.
:::

## Token Validation Patterns

### Double Submit Cookie Pattern (Default)

By default, the middleware generates and stores tokens using the `fiber.Storage` interface. These tokens are not linked to any particular user session, and they are validated using the Double Submit Cookie pattern.  The token is stored in a cookie, and then sent as a header on requests. The middleware compares the cookie value with the header value to validate the token. This is a secure pattern that does not require a user session.

When the authorization status changes, the previously issued token MUST be deleted, and a new one generated. See [Token Lifecycle](#token-lifecycle) [Deleting Tokens](#deleting-tokens) for more information.

:::caution
When using this pattern, it's important to set the `CookieSameSite` option to `Lax` or `Strict` and ensure that the Extractor is not `FromCookie`, and KeyLookup is not `cookie:<name>`.
:::

:::note
When using this pattern, this middleware uses our [Storage](https://github.com/gofiber/storage) package to support various databases through a single interface. The default configuration for Storage saves data to memory. See [Custom Storage/Database](#custom-storagedatabase) for customizing the storage.
:::

### Synchronizer Token Pattern (with Session)

When using this middleware with a user session, the middleware can be configured to store the token within the session. This method is recommended when using a user session, as it is generally more secure than the Double Submit Cookie Pattern.

When using this pattern it's important to regenerate the session when the authorization status changes, this will also delete the token. See: [Token Lifecycle](#token-lifecycle) for more information.

:::caution
Pre-sessions are required and will be created automatically if not present. Use a session value to indicate authentication instead of relying on the presence of a session.
:::

## Defense In Depth

When using this middleware, it's recommended to serve your pages over HTTPS, set the `CookieSecure` option to `true`, and set the `CookieSameSite` option to `Lax` or `Strict`. This ensures that the cookie is only sent over HTTPS and not on requests from external sites.

:::note
Cookie prefixes `__Host-` and `__Secure-` can be used to further secure the cookie. Note that these prefixes are not supported by all browsers and there are other limitations. See [MDN#Set-Cookie#cookie_prefixes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#cookie_prefixes) for more information.

To use these prefixes, set the `CookieName` option to `__Host-csrf_` or `__Secure-csrf_`.
:::

## Referer Checking

For HTTPS requests, this middleware performs strict referer checking. Even if a subdomain can set or modify cookies on your domain, it can't force a user to post to your application, since that request won't come from your own exact domain.

:::caution
When HTTPS requests are protected by CSRF, referer checking is always carried out.

The Referer header is automatically included in requests by all modern browsers, including those made using the JS Fetch API. However, if you're making use of this middleware with a custom client, it's important to ensure that the client sends a valid Referer header.
:::

## Token Lifecycle

Tokens are valid until they expire or until they are deleted. By default, tokens are valid for 30 minutes, and each subsequent request extends the expiration by the idle timeout. The token only expires if the user doesn't make a request for the duration of the idle timeout.

### Token Reuse

By default, tokens may be used multiple times. If you want to delete the token after it has been used, you can set the `SingleUseToken` option to `true`. This will delete the token after it has been used, and a new token will be generated on the next request.

:::info
Using `SingleUseToken` comes with usability trade-offs and is not enabled by default. For example, it can interfere with the user experience if the user has multiple tabs open or uses the back button.
:::

### Deleting Tokens

When the authorization status changes, the CSRF token MUST be deleted, and a new one generated. This can be done by calling `handler.DeleteToken(c)`.

```go
handler := csrf.HandlerFromContext(ctx)
if handler != nil {
    if err := handler.DeleteToken(app.AcquireCtx(ctx)); err != nil {
        // handle error
    }
}
```

:::tip
If you are using this middleware with the fiber session middleware, then you can simply call `session.Destroy()`, `session.Regenerate()`, or `session.Reset()` to delete the session and the token stored therein.
:::

## BREACH

It's important to note that the token is sent as a header on every request. If you include the token in a page that is vulnerable to [BREACH](https://en.wikipedia.org/wiki/BREACH), an attacker may be able to extract the token. To mitigate this, ensure your pages are served over HTTPS, disable HTTP compression, and implement rate limiting for requests.
