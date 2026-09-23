---
name: controller-advice-style
description: The house style for error responses of a Spring MVC REST application - `@RestControllerAdvice`, `@ExceptionHandler`, `ProblemDetail` (RFC 9457), `ResponseEntityExceptionHandler`, `AuthenticationEntryPoint`, `AccessDeniedHandler`. Load it before writing or changing any of them, when a new exception has to reach the client, when shaping validation `errors`, and when an endpoint returns 403 where 401 was expected, a `@PreAuthorize` denial comes back as 500, a status is lost to a catch-all handler, one module's exception is answered by another module's advice, or an error body has its fields nested under `properties`. Not for the controllers themselves (the separate `controller-style` skill) and not for documenting errors in Swagger (the separate `swagger-style` skill).
---

# Controller advice style

House rules for how an exception leaves a REST application as an error response. An advice or handler that breaks
one of them is either wrong, or a reason to change the rule here deliberately.

The controllers that throw follow the `controller-style` skill
([../controller-style/SKILL.md](../controller-style/SKILL.md)): a controller never catches, so every failure
arrives here as an exception.

## The advice

**Every error response is a `ProblemDetail`** (RFC 9457) from an `@ExceptionHandler` of a
`@RestControllerAdvice` - never a body of the project's own shape, never `ResponseEntity<String>`. Client
libraries already read the standard one.

**Who answers what:**

| Exception | Answered by |
|---|---|
| standard Spring MVC - no handler, method not supported, `415`, `406`, a missing parameter, an unreadable body, a type mismatch, method validation | `ResponseEntityExceptionHandler`, which the advice extends. Only a `handle*` whose response is shaped differently is overridden - validation adding `errors`; a handler of one's own for each loses the statuses nobody remembered |
| `AccessDeniedException`, `AuthenticationException` | Spring Security's `ExceptionTranslationFilter`: an `AuthenticationEntryPoint` `401`, an `AccessDeniedHandler` `403`, both through the same factory as the advice - which hands them back ([security failures](#security-failures)) |
| a domain exception | its own handler. A condition the client can cause, such as a unique constraint, becomes a domain exception in the service - `409` with its own type - never a mapping of `DataIntegrityViolationException` in the advice |
| JDK and infrastructure - `IllegalArgumentException`, `IllegalStateException`, a lost connection | no handler of its own: a bug or an outage, it falls into the catch-all `500` |
| everything else | the catch-all on `Exception`, never `Throwable` - an `Error` is the JVM failing, not an answer |

A standard exception keeps Spring's status and title and the type `about:blank` - RFC 9457's "the status says it
all", true of a `405` or a `415`; a project type is given only where the advice shapes the response anyway.

**`path` and `timestamp` reach Spring's responses through `handleExceptionInternal`**, which every inherited
`handle*` passes through. The global advice overrides that one method, stamps both and logs:

```java
@Override
protected ResponseEntity<Object> handleExceptionInternal(Exception exception, @Nullable Object body,
        HttpHeaders headers, HttpStatusCode statusCode, WebRequest request) {
    if (body instanceof ProblemDetail problem) {
        Problems.stamp(problem, ((ServletWebRequest) request).getRequest().getRequestURI(), clock.instant());
    }
    return super.handleExceptionInternal(exception, body, headers, statusCode, request);
}
```

**An advice is scoped to the controllers it serves** - `@RestControllerAdvice(basePackageClasses =
OrderController.class)`. A global one is right only where its exceptions are thrown under controllers its owner
did not write (a library's, inside the application's endpoints), and as the shared advice of a multi-module
application ([several modules](#several-modules)).

**A handler is `handle<ExceptionName>`**, takes the exception and the `HttpServletRequest`, logs, and returns what
the factory builds - it assembles no `ProblemDetail`. No overloading: the name is what a stack trace and a test
refer to. Exempt are `handleSecurityException` below, and an overridden `handle*` of
`ResponseEntityExceptionHandler`, which keeps Spring's name and signature, `WebRequest` included.

**The advice takes a `Clock`** through its constructor and stamps `clock.instant()`, so a test can fix the time.

| Failure | Logged at | How |
|---|---|---|
| `5xx` | `error` | with the exception as the last argument - the response carries a fixed sentence, the log is the only place the cause survives |
| `4xx` | `debug` | one line, type and request URI, no stack trace - `warn` floods the log and trips alerts over requests nobody here can fix; `debug` costs nothing and can be switched on |

## Security failures

**The advice holding the catch-all hands security exceptions back.** A `@PreAuthorize` denial is thrown inside
the `DispatcherServlet`, under the advice, so the catch-all takes it before Spring Security sees it - `500` for
the anonymous caller and for the one without the role alike. Within one advice the closest type wins over
`Exception`, so this is asked first:

```java
@ExceptionHandler({AccessDeniedException.class, AuthenticationException.class})
public void handleSecurityException(Exception exception) throws Exception {
    throw exception;
}
```

**A `403` where `401` was due is a chain without an entry point.** `oauth2ResourceServer` and `httpBasic` bring
one, a JWT filter of the project's own does not, and Spring Security falls back to `Http403ForbiddenEntryPoint`.
The chain sets its entry point in `exceptionHandling` explicitly, whatever the mechanism.

**Outside the `DispatcherServlet` the response is written by hand**: no content negotiation there, so the entry
point, the access-denied handler or a filter sets the status and `Content-Type: application/problem+json` itself,
and serialises with the injected `ObjectMapper` bean - Boot's carries the mixin that writes properties at the top
level; `new ObjectMapper()` writes `"properties": {"path": ...}` and every unset field as `null`.

**A failed login is one `401` with one detail** for an unknown user and a wrong password alike - two answers tell
anyone which accounts exist.

## The problem response

**Status, type and title are decided in one place** - a package-private factory, one method per kind of failure,
called by the advice and by everything else that writes an error (a filter, the entry point, the access-denied
handler), which therefore live in its package. It is the table "what went wrong -> status, type, title";
nothing else in the code states any of the three.

```java
static ProblemDetail orderAlreadyShipped(String detail, String instance, Instant timestamp) {
    return Problems.problem(HttpStatus.CONFLICT, ProblemTypes.ORDER_ALREADY_SHIPPED, "Order already shipped",
            detail, instance, timestamp);
}
```

**The type is a `ProblemTypes` constant**, an absolute URI `https://<project>/errors/<kebab-case-name>` - a name,
nothing is fetched from it. It is the only field a client may branch on, so it is public contract: renaming one
breaks every client that does. One type always has the same status and title.

**The fields**, and nothing beyond them unless the failure has something the client acts on (the id of the
conflicting resource):

| Field | Value |
|---|---|
| `status` | from the factory; Spring's for a standard exception |
| `type` | a `ProblemTypes` constant; `about:blank` for a standard exception |
| `title` | fixed per type, a short phrase in sentence case |
| `detail` | what happened in this occurrence, for a human |
| `instance` | `request.getRequestURI()` |
| `path` | a property, the same URI - repeated on purpose, for clients of Spring Boot's default error body |
| `timestamp` | a property, `clock.instant()` |

**Only a validation failure adds `errors`**, at the top level: `errors: [{field, message}]`. `field` is the name
in the JSON (the `@JsonProperty` one), not the Java field; a rule across fields gives one entry per field with the
same message. `setProperty("errors", ...)` already writes at the top level - a map of one's own around it
(`properties: {errors: [...]}`) nests it where no client looks. A failure not about a field gets no pseudo-field
(`server`, `url`, `data`): `type` and `detail` say what happened.

**A `4xx` detail says what the client did wrong** and may carry the exception's message, written for that.
**A `5xx` detail is a fixed sentence** - `The operation could not be completed` - never the message or the cause,
which carry SQL, table names and stack fragments to the caller; the cause goes to the log.

**One handler per type** - exceptions of one type share a handler through a common supertype; exceptions of
different types never do, even with the same status.

**Localisation goes through Spring's message codes**: with a `MessageSource` on the
`ResponseEntityExceptionHandler`, `problemDetail.title.<exception class>` and `problemDetail.<exception class>`
resolve the title and the detail in the request's locale. `type` is never localised - a client branches on it.
Without localisation both are English, and translating is the client's business.

## Several modules

**One global advice and one small advice per module.** The global one lives in the shared module and handles the
framework exceptions, the common ones and the catch-all, at `@Order(Ordered.LOWEST_PRECEDENCE)`. A module's
advice is scoped to its packages, handles only its own domain exceptions and must carry an `@Order` ahead of
it - `LOWEST_PRECEDENCE - 1`. Without one it ties with the global advice, the tie falls to whichever bean
registered first, and Spring takes the first advice with any matching handler - the global catch-all then answers
the module's exception with `500`.

**No handler is declared in two advices**, and no base class has its handlers redeclared in every module only to
call `super` - an `@ExceptionHandler` is inherited, and the copies drift apart.

**The one place for status, type and title becomes one per module.** A module cannot reach the shared module's
package-private factory, and a public one would let every module restate the shared failures. So the shared
module holds a public builder, `Problems.problem(status, type, title, detail, instance, timestamp)`, which sets the
fields and stamps `path` and `timestamp` but decides none of them, and its own factory for the failures all
modules share - validation, `401`, `403`, `500`. Each module keeps a package-private factory beside its advice
(`LoanProblems`) with its own `ProblemTypes` (`LoanProblemTypes`), on the same builder; no two modules declare the
same type name.
