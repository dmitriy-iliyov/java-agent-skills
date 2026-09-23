---
name: swagger-style
description: The house style for springdoc-openapi annotations on a Spring REST controller and its DTOs - `@Operation`, `@ApiResponses`, `@Parameter`, `@Schema`, `@ParameterObject`. Load it before adding or changing `io.swagger.v3.oas.annotations` on a controller, a record or an enum, when a new endpoint is added to an annotated controller, when checking `/v3/api-docs` or Swagger UI against the code, and when Swagger UI documents an error as returning the success body, shows a `@ModelAttribute` filter as one object parameter, asks for a parameter the server fills in itself, answers 415 on a multipart "Try it out", or drops enum descriptions. Not for javadoc (the separate `javadoc-style` skill), and not for designing paths, statuses or error responses - those are the `controller-style` and `controller-advice-style` skills, taken as given here.
---

# Swagger style

House rules for springdoc-openapi annotations on a REST controller and its DTOs. An annotation that breaks one
of them is either wrong, or a reason to change the rule here deliberately.

Every string in an annotation is English. The annotations document decisions made elsewhere and make none of
their own: paths, verbs, statuses and whether `@ResponseStatus` is written at all follow the `controller-style`
skill ([../controller-style/SKILL.md](../controller-style/SKILL.md)); which errors an endpoint answers and what an
error response carries follow the `controller-advice-style` skill
([../controller-advice-style/SKILL.md](../controller-advice-style/SKILL.md)).

## What gets annotated

| Where | What it carries |
|---|---|
| **Endpoint method** | `@Operation(summary = ...)` and `@ApiResponses` - see [Responses](#responses) for which codes |
| **Path variable, query parameter** | `@Parameter(description = ...)`; a path variable also `required = true` |
| **Request / response record** | `@Schema(description = ...)` on the type and on every component - see [Fields](#fields) |
| **Enum used in a request or response** | `@Schema(description = ..., enumAsRef = true)` on the type, nothing on the constants - see [Fields](#fields) |
| **Query parameters bound through `@ModelAttribute`** | `@ParameterObject` on the method parameter - see [Traps](#traps) |
| **Type from a module without a springdoc dependency** | nothing, referenced through `implementation` - its fields show without descriptions, accepted rather than pulling `swagger-annotations` into a module with no web layer |

`@Operation` carries a `summary` and nothing else unless the behaviour cannot fit in one line. The annotations
sit above the mapping annotation in the order `@Operation`, `@ApiResponses`, `@XxxMapping`, `@ResponseStatus`:

```java
@Operation(summary = "Delete order by id")
@ApiResponses({
        @ApiResponse(responseCode = "204", description = "Order successfully deleted"),
        @ApiResponse(responseCode = "404", description = "Order not found", content = @Content)
})
@DeleteMapping("/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT)
public void delete(@Parameter(description = "Id of the order to delete", required = true)
                   @PathVariable("id") UUID id) {
```

## Wording

No summary, description or parameter text ends with a period, a full sentence such as an enum's description
included: `All requested entities were successfully processed`. An `example` is exempt - it is a value the
endpoint returns, copied with whatever punctuation the code gives it.

**Summary** - an imperative verb phrase in sentence case, naming the resource and what selects it: `Get order by
id`, `Update status for multiple orders by ids or customer`, `Count orders by status and/or customer`. What an
endpoint does when optional parameters are all absent goes in parentheses at the end: `(counts all orders if no
parameters are provided)`.

**Response description** - fixed forms, so that the UI reads the same across endpoints:

| Status | Form |
|---|---|
| 2xx, one resource | `<Resource> successfully <retrieved / updated / deleted / created>` |
| 2xx, a command (`/{id}/<verb>`) | `<Resource> successfully <the verb's past participle>`: `Order successfully cancelled` |
| 2xx, a list or a page by a query | `<Resource> list successfully retrieved` |
| 2xx, a batch or a count | `<Resource> batch successfully <...>`, `<Resource> count successfully retrieved` |
| 404 | `<Resource> not found` |
| 400 | the cause, in the terms the caller used: `Request validation failed`, `Order is already shipped` |

A code appears once in `@ApiResponses`; several causes for one code are joined with `or`: `Request validation
failed or order is already shipped`.

**Parameter description** - `Id of the <resource>` for a path variable, extended with the purpose where the
method changes something: `Id of the order to update`. A query filter is `Filter by <field>`: `Filter by order
status`, `Filter by customer`.

**Field description** - a noun phrase saying what the value is for: `Number of orders per page`, `New status to
assign to all specified orders`.

Throughout: `id` in lowercase (`Id` at the start of a string), abbreviations in capitals (`API`, `UUID`), an
enum constant as it is spelled in code (`PENDING_PAYMENT`).

## Responses

**The documented code is the code the method returns.** `@ApiResponse(responseCode = "204")` over a method that
returns a body with `200` is a lie the UI repeats to every caller.

**Listed is what the endpoint produces on purpose and the caller can act on**, not every status that can
physically come back:

- `404` wherever a lookup by id can miss;
- `400` wherever a body or a parameter is validated - `@Valid`, an enum bound from a string, a constraint across
  fields;
- `400` / `409` wherever the service refuses the state the resource is in.

A value that fails only its type's conversion - a malformed `UUID` or number in the path or the query - is not a
`400` of its own: the type in the schema already says what is accepted. An enum is listed although it too fails
by conversion, because its constants are a choice the caller makes. So `DELETE /orders/{id}` lists `204` and
`404`, while `GET /orders/count?status=` lists a `400` for the status.

Never a `500`, nor anything the advice maps from an infrastructure failure (`DataAccessException` and the like):
no endpoint produces those by its own rules, and no change to the request avoids them. A cause the endpoint does
produce and `@ApiResponses` leaves out is a gap, not a simplification.

**`content` follows the shape of the body:**

```java
content = @Content(schema = @Schema(implementation = Order.class))                        // one object
content = @Content(array = @ArraySchema(schema = @Schema(implementation = Order.class)))  // a list
content = @Content(schema = @Schema(implementation = Long.class))                         // a scalar
```

**Every response that is not the body of the method carries `content = @Content`** - an error response, and a
`204` of a method whose return type is not `void`. With no `content` at all springdoc fills it in from the
return type, so a `404` over `Order get(...)` is documented as returning an `Order`; the empty `@Content`, as in
the example above, keeps it empty.

## Fields

Every component of a request or response record has `description`, `example` and `requiredMode`:

```java
@Schema(
        description = "Number of orders per page",
        example = "50",
        minimum = "10",
        maximum = "100",
        requiredMode = Schema.RequiredMode.REQUIRED
)
@Min(value = 10, message = "pageSize must be at least {value}")
@Max(value = 100, message = "pageSize must be at most {value}")
int pageSize
```

**The example is a value the endpoint would actually accept or return**, never `"string"` or `"test"`, and it
satisfies the component's own constraints: inside the `@Min` / `@Max` bounds, a well-formed UUID, a constant that
exists in the enum, a collection no larger than its `@Size`.

**`requiredMode` is stated on every component, a primitive included** - a primitive without it is rendered as
optional, and the caller cannot tell whether `pageSize` may be left out. `REQUIRED` goes with `@NotNull` /
`@NotBlank` / `@NotEmpty` on a reference type; the two say the same thing and must not disagree. A primitive
carries none of them, so its type's default (`0`, `false`) decides: `REQUIRED` where the default fails the
component's own constraints (`pageSize` under `@Min(10)`), `NOT_REQUIRED` where it passes (`page` under
`@Min(0)`).

**A numeric bound in `@Schema` repeats the validation annotation exactly** - `minimum` / `maximum` with `@Min` /
`@Max`. Not on a collection: `maximum` is a numeric keyword and on a `Set` or a `List` gives a client nothing to
check. A collection's size comes from `@Size(max = ...)` alone, which springdoc renders as `maxItems` (and
`minItems`) by itself - no `@Schema` or `@ArraySchema` attribute repeats it.

**A constraint across fields is stated on every field it involves.** "Either `ids` or `customerId`, not both"
cannot be expressed in the schema, and a caller who sees two optional fields will send both. Each says it: `List
of order ids to delete, not together with customerId`, `Customer whose orders to delete, not together with ids`.

**An enum is described on its type, with `enumAsRef = true`.** swagger-core does not read `@Schema` on an enum
constant; a description there appears nowhere. Without `enumAsRef` the enum is inlined into every field that uses
it and the field's description replaces the type's; with it the enum is a schema of its own under `components`
and the field keeps its description next to the `$ref`. Where a constant's meaning does not follow from its
name, the type's description gives it, in the same words as the constant's javadoc:

```java
@Schema(description = "Payment state: PENDING_PAYMENT - waiting for payment, PAID - paid in full",
        enumAsRef = true)
public enum PaymentStatus {
```

## Traps

**A record bound through `@ModelAttribute` is shown as one parameter.** Without `@ParameterObject`,
`@ModelAttribute @Valid OrderFilter filter` becomes a single required query parameter `filter` of type `object`,
and Swagger UI has no field for `status` or `pageSize`. With it each component is a query parameter of its own,
with its `@Schema` description and bounds:

```java
public List<Order> list(@ParameterObject @ModelAttribute @Valid OrderFilter filter) {
```

`springdoc.default-flat-param-object=true` does the same application-wide - fine where the application owns its
configuration; a controller in a library or a starter cannot count on it and carries `@ParameterObject` itself.

**A method of a record leaks into the schema as a property.** Any `is*` / `get*` method is a bean getter, rendered
as a property the client is invited to send - `isValid()` becomes `valid: {type: boolean}`, `isExactlyOneTarget()`
becomes `exactlyOneTarget`. So a rule across fields is a class-level constraint, as `controller-style` requires,
not an `@AssertTrue` method; any other helper method on a request or response record gets `@Schema(hidden = true)`:

```java
@Schema(hidden = true)
public boolean isByCustomer() {
```

**A parameter the server fills in is shown as one the client has to send.** springdoc leaves out the ones it
recognises - `@AuthenticationPrincipal` (a meta-annotation over it included), `Locale`, `HttpServletRequest`,
`Principal` - but not one bound by the application's own `HandlerMethodArgumentResolver`: `@ClientIp String
clientIp` becomes a required query parameter the UI asks the caller to type in. The annotation the resolver
reads carries `@Parameter(hidden = true)` once, instead of every parameter repeating it:

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Parameter(hidden = true)
public @interface ClientIp {
}
```

Where that annotation lives in a module without a springdoc dependency, the documentation's configuration calls
`SpringDocUtils.getConfig().addAnnotationsToIgnore(ClientIp.class)` instead.

**A JSON part of a multipart request is documented without its type.** springdoc renders `@RequestPart("report")
DamageReportRequest report` as an object property of the `multipart/form-data` schema with no `encoding`, so
nothing tells a client the part goes as `application/json`. Spring reads a part by its own `Content-Type`: sent
with none or as `text/plain`, it is a `415`, and the endpoint cannot be called from Swagger UI at all. The
encoding goes in a Swagger `@RequestBody`, fully qualified next to Spring's, after `@ApiResponses`, with no
`schema` of its own - springdoc merges it with the one it infers from the parameters:

```java
@io.swagger.v3.oas.annotations.parameters.RequestBody(content = @Content(
        mediaType = MediaType.MULTIPART_FORM_DATA_VALUE,
        encoding = @Encoding(name = "report", contentType = MediaType.APPLICATION_JSON_VALUE)))
@PostMapping(path = "/{id}/damage-reports", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
```

A file part (`MultipartFile`) needs no encoding - it is rendered as `format: binary` and sent with the file's type.

## Checking the result

Nothing compiles or tests the annotations, so the result is checked in the generated document, not only in the
source: `/v3/api-docs` (or Swagger UI) where the application can be started, otherwise the schema of one type
straight from swagger-core:

```java
Json.pretty(ModelConverters.getInstance(true).readAll(OrderFilter.class).get("OrderFilter"))
```

Compare against the code:

- each 2xx code against `@ResponseStatus` or its absence ([Responses](#responses));
- `required` against `requiredMode` and `@NotNull`; `minimum` / `maximum` / `maxItems` against the validation
  annotations ([Fields](#fields));
- no `content` on an error response - an `Order` under a `404` ([Responses](#responses));
- no property that is not a record component (`valid`), no query parameter of type `object`, no parameter the
  client does not send (`clientIp`, `user`), an `encoding` on every JSON part ([Traps](#traps));
- every summary and description read as text. A rename by the IDE or by search-and-replace rewrites the word
  inside the strings too - `count` to `countActive` turns `Customer to count orders for` into `Customer to
  countActive orders for` - and a summary with a missing bracket renders as it is.
