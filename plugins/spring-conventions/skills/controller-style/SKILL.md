---
name: controller-style
description: The house style for Spring MVC REST controllers - `@RestController`, `@RequestMapping`, `@GetMapping` / `@PostMapping`. Load it before writing or rewriting a controller or an endpoint, when choosing a URL, an HTTP verb or a response status for one, when binding path variables, query parameters, a `@RequestBody`, a `@ModelAttribute` filter or a multipart `@RequestPart`, when adding a request record or its `@Valid` constraints, when injecting services into a controller, placing `@PreAuthorize` or getting the current user through `@AuthenticationPrincipal`, when deciding between returning a body and `ResponseEntity`, and when judging whether a controller is thin enough. Not for exception handling and error responses (the separate `controller-advice-style` skill), not for Swagger annotations (the separate `swagger-style` skill), not for the service layer, and not for server-side rendering with views.
---

# Controller style

House rules for Spring MVC controllers. A controller that breaks one of them is either wrong, or a reason to
change the rule here deliberately.

[Any controller](#any-controller) holds for every controller, [REST](#rest) for `@RestController` and
`@RestControllerAdvice` only; a `@Controller` returning a view follows the first part and nothing more is settled
for it. Swagger annotations follow the `swagger-style` skill ([../swagger-style/SKILL.md](../swagger-style/SKILL.md)),
which documents the decisions made here - a status chosen here is the status listed there. What happens to an
exception once it leaves a controller - the advice, the error body, `401` and `403` - is the
`controller-advice-style` skill ([../controller-advice-style/SKILL.md](../controller-advice-style/SKILL.md)).

## Any controller

### Thin

**A controller is as thin as it can be made** - an adapter between transport and service that decides nothing.
Every line that is not binding, delegating or returning is logic no test of the service will ever see.

**A method is one call to the service and nothing else** - no branching on the result, no loop, no mapping
between domain types, no transaction, no logging:

```java
@PostMapping("/{id}/approve")
public Order approve(@CurrentUser UserPrincipal user, @PathVariable("id") UUID id) {
    return service.approve(user.accountId(), id);
}
```

Choosing an implementation is logic too: a registry of strategies, the lookup and the error for a missing key
live in a service or an orchestrator - `manualTaskService.run(type)`, not `creators.get(type).create()`. The one
exception is **reference data that is a constant of the code** - an enum's values, a map built from them - which
the controller may produce with `values()` and a stream: a service would only forward the same line.

**A controller never catches an exception.** A `try` / `catch` is a second, unsynchronised place deciding what a
failure looks like; the failure leaves as an exception and the advice answers it.

**Nothing unfinished is written** - no empty controller (it is still a bean), no commented-out endpoint (git keeps
history), no `System.out`, no hardcoded stand-in such as a fake user id (an endpoint that works and lies). An
endpoint that has to exist before it is finished throws, and the client gets `501`. This governs code being
written; placeholders already in a codebase are not removed as a side effect of other work.

### What reaches the service

**What the client sent and what the server vouches for travel apart.** The request record reaches the service as
it arrived - immutable, no setters - and the path id and the current user are separate arguments:
`service.update(user.accountId(), id, dto)`. `dto.setOwnerId(...)` mixes the two, and a settable owner is one
mapping bug away from trusting the client's. Too many arguments for one call become an immutable command -
`new UpdateOrderCommand(accountId, id, dto)`.

**Servlet types do not reach a domain service.** A header or the locale is taken as `@RequestHeader` or `Locale`
and passed as a value. Work that is itself web work - a cookie, ending a session - lives in a web-layer facade,
which may take `HttpServletRequest` / `HttpServletResponse`; a domain service never does.

### Dependencies

**Few, and a facade keeps them few** - one facade over several services, not each of them. A `@Qualifier` is
allowed where a specific implementation, a retrying decorator say, has to be chosen.

**A controller is split only for a reason** - past about twenty methods, two domain areas, or several audiences
(see [URLs](#urls)). Not otherwise: three small controllers over one resource are harder to find than one.

**Constructor injection, `private final` fields, no `@Autowired`:**

- **in an application**, Lombok's `@RequiredArgsConstructor`, no null check - a bean is never `null`. A
  `@Qualifier` on a field needs `lombok.copyableAnnotations += org.springframework.beans.factory.annotation.Qualifier`
  in `lombok.config`, or Lombok drops it and Spring injects the wrong bean or fails on two candidates;
- **in a library or a starter**, a hand-written constructor with
  `Objects.requireNonNull(service, "service cannot be null")` - someone else's code may construct it. The
  controller and its advice are not component-scanned but registered as `@Bean` in the auto-configuration under
  `@ConditionalOnMissingBean`, so the application can replace either.

### The current user and authorisation

**Outside the module that owns authentication, one principal type is bound**, through `@AuthenticationPrincipal`
or a meta-annotation over it (`@CurrentUser`), and the parameter is named for what it is - `user`, `principal`.
The authentication module alone may bind its wider types - an access-token or refresh-token principal.

**`@PreAuthorize` goes where it says the least** - a rule shared by every method on the class, a method's own only
where it differs. Where most methods would differ, each carries its own and the class none: a class rule
overridden by half its methods says nothing true about the class.

**"Any logged-in user" is `isAuthenticated()`**, never `hasAnyRole(...)` listing every role - the list breaks
silently the day a role is added, and its holders cannot even log out. `hasAnyRole` is for a set chosen on purpose.

**A whole audience is also closed by URL** - `/api/v1/admin/**`, `/api/v1/system/**` get a rule in the
`SecurityFilterChain`, so a forgotten annotation opens nothing.

### Validation

**Input is validated by annotations, not by code** - `@Valid` on the `@RequestBody` or `@ModelAttribute` record,
constraints on its components.

**A rule across fields is a class-level constraint, reported on every field it involves**, so that `errors` (see
`controller-advice-style`) names fields the client sent:

```java
context.disableDefaultConstraintViolation();
for (String field : List.of("ids", "customerId")) {
    context.buildConstraintViolationWithTemplate(context.getDefaultConstraintMessageTemplate())
            .addPropertyNode(field)
            .addConstraintViolation();
}
```

Not an `@AssertTrue` method: its violation carries the method's property name (`isExactlyOneTarget()` reports
`exactlyOneTarget`), which is no field of the JSON, and the method leaks into the OpenAPI schema.

**A single path variable or query parameter may carry its own constraints** - `@PathVariable("id") @Positive Long
id`, `@RequestParam("code") @Pattern(regexp = "^\\d{6}$") String code`. Spring validates them without `@Validated`
on the class (which would validate twice) and reports `HandlerMethodValidationException`. More than two such
parameters on one method are a record bound through `@ModelAttribute`.

**A body is always a record** - never a bare `String` with constraints, never a `Map`: a map carries no
constraints, has no OpenAPI schema, and reads a mistyped key as `null` instead of failing with `400`. A response
is never a `Map` or a hand-built JSON `String` either; a list of records or a scalar is fine.

**The one exception is JSON Merge Patch** (RFC 7396), where an absent field is left alone and `null` removes it -
a record cannot tell the two apart, so the body is a `JsonNode` with `consumes = "application/merge-patch+json"`.
The patch is applied to a DTO of the fields a client may change, never to the entity (which would let it rewrite
the id, the status, the counters); the merged DTO goes through the `Validator` before anything is saved; the merge
lives in the service.

**A validation message is `<field> must <condition>`** - no trailing period, no exclamation mark, a bound
interpolated rather than typed: `comment must be at most {max} characters`. A typed number goes stale.

**Every binding annotation names its parameter** - `@PathVariable("id")`, `@RequestParam(value = "status",
required = false)`. The compiled parameter name is gone the moment the code is built without `-parameters`.

## REST

### URLs

**Every endpoint carries its version right after `/api`** - `/api/v1/...`, authentication, CSRF and every
audience included. Versioning some endpoints is worse than either scheme; one segment everywhere is what
`ApiVersionConfigurer.usePathSegment(1)` reads, and the whole API moves to `v2` together.

**An audience gets its own prefix and its own controller.** The prefix follows the version and maps one to one to
a role, since the `SecurityFilterChain` closes it:

| Prefix | Audience |
|---|---|
| `/api/v1/...` | the public API and ordinary users |
| `/api/v1/admin/...` | administrators |
| `/api/v1/system/...` | services - monitoring such as Prometheus, internal services of a distributed system |
| `/api/v1/worker/...` | workers only - parsers, crawlers, server functions that take tasks and report back |

A service that merely calls the API is `system`, not `worker`. One controller per audience over the same
resource - `AdminOrderController`, `WorkerTaskController` - so no endpoint inherits another audience's
authorisation.

**The current user's resources, by meaning:** `/api/v1/me/<resources>` for a collection the user owns
(`/me/orders`), `/api/v1/<resources>/me` where the user is the resource (`/profiles/me`). Never
`/<resources>/me/<other-resources>`.

**Casing**: path segments in kebab-case (`/password-recovery`); query parameters and path variables in the casing
of the JSON body, camelCase by default (`?pageSize=`, `{orderId}`) - to the client a filter and a body field are
the same names.

**A command that is not CRUD is `POST /<resources>/{id}/<verb>`** - `POST /orders/{id}/approve`. Id first, verb
last; `POST` because it is a command, not a partial replacement of fields.

**One base path per controller, on the class, naming the resource it serves** - `@RequestMapping("/api/v1/orders")`,
with operations over many resources under its `/batch`. A sub-resource of the same domain may have a controller of
its own (`/orders/{id}/notes`, `/admin/products/import-errors`); another domain is never mounted under it -
shipments live under `/shipments`, not `/orders/shipments` because orders needed them first.

**`GET` changes nothing** - it is safe by definition, and proxies, clients and prefetchers repeat it on their own.
Taking, locking or leasing something is a command: `POST /api/v1/worker/tasks/claim`, never `GET /tasks/batch`
with a lock behind it.

**A computation is a resource if its result is kept, a verb if not** - `POST /api/v1/recognitions` answers `201`
and the history is `GET /api/v1/me/recognitions`; `POST /api/v1/images/classify` answers `200` and forgets.

**Reference data is grouped by audience** - dictionaries may share a controller only with others open to the same
audience; an admin-only one never sits with public ones.

### Parameters and bodies

**A key in the path has its domain type** - `UUID id`, `@Positive Long id`, or an enum where each constant is one
resource (`@PathVariable("jobType") JobType jobType`) - never a `String` put through `UUID.fromString`. A malformed
value becomes `MethodArgumentTypeMismatchException`, a `400`; a UUIDv7 requirement is a constraint on the `UUID`.
An enum appears in the URL as its constant's name, as in JSON (`/export-jobs/NIGHTLY_REPORT`), with no converter
of the project's own.

**The query selects, the body carries what changes.** Filter, page and sort are query parameters, and as a group
they are bound into one record with `@ModelAttribute @Valid`, not a row of `@RequestParam`. What a request sets is
a body - except a single scalar set on its own (`PATCH /tasks/{id}/status?status=DONE`).

**Multipart**: a single file and nothing else may be `@RequestParam("file") MultipartFile file`. With a JSON part,
every part is a `@RequestPart` and the JSON one is typed - `@RequestPart("order") @Valid OrderDto dto,
@RequestPart("picture") MultipartFile picture`: Spring reads it by the part's own `Content-Type`
(`application/json`) and validates it like a body. A JSON part taken as a `String` and parsed by hand is not
allowed.

**`consumes` is declared on every endpoint that does not take JSON; `produces` is never written**, and a
controller never sets a `Content-Type` - content negotiation does. Only code outside the `DispatcherServlet` sets
one (see `controller-advice-style`).

### Responses

**A method returns the body itself**, with `@ResponseStatus` only where the status is not `200`. `ResponseEntity`
is for a header or a status that depends on the call - a `Location`, a `304`, `Prefer` - never for a fixed status.

The table is the default, not a law - a situation unlike the row gets its own status:

| Operation | Returns | Status |
|---|---|---|
| read one, read a batch, count | the resource, a list, a scalar | `200` |
| create | the created resource, with `Location` | `201` |
| create a batch, all or nothing | the list of created resources, no `Location` | `201` |
| create a batch that may partly fail | a result record saying how many were created | `200` |
| create something with no address of its own - a report sent by a worker | nothing | `204` |
| update one | the resource as it is after the change | `200` |
| a command (`/{id}/<verb>`) | the resource as it is after the command | `200` |
| a command with no resource to show - logout, sending a code | nothing | `204` |
| delete one | nothing | `204` |
| update or delete a batch | a result record saying how many were processed | `200` |

`204` is a valid answer to `PUT` and `PATCH`; the resource is returned anyway so the client need not `GET` it
again to see what the server made of the change - a normalised value, a generated field, a status that moved.
**A `200` with an empty body is never returned** - so a `void`
method always carries `@ResponseStatus(HttpStatus.NO_CONTENT)` (bare, it answers exactly that empty `200`), and
never `CREATED`: a `201` names what it created.

**A client that wants no body sends `Prefer: return=minimal`** (RFC 7240) and gets `204` or `201` without one;
no header or `return=representation` gets the table. No `?returnBody=` of the project's own. The endpoint reads
`@RequestHeader(value = "Prefer", required = false)` and answers through `ResponseEntity` - the one place the
status depends on the request.
