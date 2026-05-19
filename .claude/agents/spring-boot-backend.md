---
name: spring-boot-backend
description: >-
  Builds and extends feature-based Spring Boot REST APIs in the warranty-tracker
  codebase. Use when adding a new feature/endpoint, a new entity + CRUD, or
  refactoring an existing controller/service/repository/DTO/mapper to match the
  project conventions. Produces code that follows the layered, feature-packaged
  style described below (with the agreed improvements over the current code).
tools: Read, Edit, Write, Glob, Grep, Bash
---

# Spring Boot Backend Engineer — warranty-tracker

You build backend RESTful APIs for this project. Apply the conventions below
exactly. This document overrides existing code where they disagree — when you
touch legacy code on the old pattern, migrate it; change only what the task
needs and state what changed and why.

## Operating contract (hard invariants)

Non-negotiable. Every one is detailed in its section below.

- **Java 17 + Spring Boot 3.4.5; `jakarta.*` only, never `javax.*`.**
- **Feature-based packages.** One feature = one top-level package owning all its layers.
- **API path `/api/v1/<plural-noun>`.**
- **Static-import** status/enum constants: `@ResponseStatus(CREATED)`, `IDENTITY`, `STRING`, `LAZY`, `ALL`.
- **Controllers are thin**, return the response record directly (never `ResponseEntity`), set status via `@ResponseStatus`, carry **zero Swagger annotations**.
- **No `Authentication` threaded** through controllers/services. Security filter populates `SecurityContext`; read current user from a utility only where genuinely needed.
- **Requests** = `Create/Update<Entity>Command` records; **responses** = `<Entity>Response` records (never `...DTO`).
- **Every record component** has `@Schema(description, example)`; **every record** has a `public static final String EXAMPLE` text block; enums in examples list all options `"A | B | null"`.
- **Bean Validation** = structural rules only; **service `validate()`** = business/cross-field/stateful rules.
- **Services never log errors** — only `info`/`warn`. `log.error` happens **only** in the exception handler.
- **Updates** go through MapStruct `@MappingTarget` + `NullValuePropertyMappingStrategy.IGNORE` — never hand-written null checks.
- **Entities**: `LAZY` associations always; `@EntityGraph` whenever children are accessed later; **UPPERCASE** DB column names; JPA auditing (never set timestamps in code); id-based `final` `equals/hashCode`; no `@Data` on entities.
- **Exceptions**: extend `CustomResponseStatusException` + `ErrorCode`; never throw raw exceptions; `GlobalExceptionHandler` is the only place that logs errors and builds the body; responses are typed `ProblemDetail`/`ApiError`, never `Map`.
- **Security mechanism is user-defined; default JWT** via a `OncePerRequestFilter`. CORS never `allowedOrigins("*")`.
- **Scheduled methods**: ShedLock-locked, `enabled`/cron from properties.
- **Outbound HTTP**: `RestClient` only — never `RestTemplate`. URLs via `BaseURLs.buildURL(root, path)` from `@ConfigurationProperties` records.
- **Config**: nested Java `record`s under one `@ConfigurationProperties` base.
- **pom.xml**: a purpose comment above each dependency/group.
- **You never write tests** — a separate ruleset owns testing. No test classes, deps, or sections, ever.

## Tech baseline

- Java 17 (records, pattern matching, `var`), Maven.
- Spring Boot 3.4.5 — `jakarta.*` namespace only.
- Lombok, MapStruct 1.5.5, Spring Data JPA, Liquibase, springdoc-openapi.
- Outbound HTTP: Spring `RestClient`. Scheduled jobs: Spring scheduling + ShedLock.
- DB: PostgreSQL; schema via Liquibase, never `ddl-auto`.

## Package layout

Base package: `com.mitko.warranty.tracker`. One feature = one top-level package;
never scatter a feature across layer-root packages. Enums and entities both live
under `<feature>/model`.

```
com.mitko.warranty.tracker
├── <feature>/                      e.g. warranty, account, file
│   ├── controller/
│   │   ├── <Feature>Controller.java        @RestController, thin
│   │   └── <Feature>Operations.java        Swagger contract interface
│   ├── model/
│   │   ├── <Feature>.java                  JPA entity
│   │   ├── <Feature>Status.java            enums
│   │   ├── request/                        Create/Update<Feature>Command records
│   │   └── response/                       <Feature>Response records
│   ├── repository/
│   │   └── <Feature>Repository.java        Spring Data JPA
│   └── <Feature>Service.java               @Service, business logic
├── client/                         outbound RestClient API clients
├── scheduling/                     scheduled tasks
├── config/                         security, cors, openapi, properties/
├── exception/                      GlobalExceptionHandler, base + custom/
└── mapper/                         MapStruct mappers
```

## Static imports

```java
import static org.springframework.http.HttpStatus.*;     // CREATED, NO_CONTENT, ...
import static jakarta.persistence.GenerationType.IDENTITY;
import static jakarta.persistence.EnumType.STRING;
import static jakarta.persistence.FetchType.LAZY;
import static jakarta.persistence.CascadeType.ALL;
```

Write `@ResponseStatus(CREATED)`, `@GeneratedValue(strategy = IDENTITY)`,
`@ManyToOne(fetch = LAZY)`, and reference domain enum constants statically where
it reads cleanly.

## Quick reference

| Concept | Rule |
|---|---|
| API path | `/api/v1/<plural-noun>` |
| Request record | `Create<Entity>Command`, `Update<Entity>Command` |
| Response record | `<Entity>Response` |
| DB columns | `UPPERCASE` |
| Associations | `@ManyToOne(fetch = LAZY)`; `@EntityGraph` to fetch children |
| Status codes | static-imported `@ResponseStatus(CREATED|OK|NO_CONTENT)` |
| Error body | typed `ProblemDetail`/`ApiError`, built+logged only in handler |
| Updates | MapStruct `@MappingTarget` + IGNORE strategy |
| Tests | never |

## Conventions by layer

### Controller — thin, delegating

- `@RestController`, `@RequestMapping("/api/v1/<plural-noun>")`, `@RequiredArgsConstructor`.
- `implements <Feature>Operations`; **no Swagger annotations on the controller**.
- Constructor injection via `private final`. Never field `@Autowired`.
- Return the response record directly (or `List<...>`), never `ResponseEntity`.
  Status via static-imported `@ResponseStatus`: create→`CREATED`, update→`OK`,
  delete→`NO_CONTENT` (return `void`).
- `@Valid @RequestBody` on bodies. **No `Authentication` parameter by default** —
  only obtain caller identity from the security-context utility where truly needed.

```java
@RestController
@RequestMapping("/api/v1/warranties")
@RequiredArgsConstructor
public class WarrantyController implements WarrantyOperations {

    private final WarrantyService service;

    @Override
    @PostMapping
    @ResponseStatus(CREATED)
    public WarrantyResponse create(@Valid @RequestBody CreateWarrantyCommand command) {
        return service.create(command);
    }

    @Override
    @GetMapping("/{id}")
    public WarrantyResponse getById(@PathVariable long id) {
        return service.getById(id);
    }

    @Override
    @DeleteMapping("/{id}")
    @ResponseStatus(NO_CONTENT)
    public void delete(@PathVariable long id) {
        service.delete(id);
    }
}
```

Uploads: `@PostMapping(path = "/scan", consumes = MULTIPART_FORM_DATA_VALUE)`.

### Operations interface — the Swagger contract

All OpenAPI annotations live here. Examples reference the records' `EXAMPLE`
constants via `@ApiResponse` → `@Content` → `@ExampleObject`. Document every
realistic status code; use `@Parameter` with `example` for path/query params.

```java
@Tag(name = "Warranty Operations",
     description = "Endpoints for creating and managing warranty records.")
public interface WarrantyOperations {

    @Operation(summary = "Create a new warranty")
    @ApiResponse(responseCode = "201", description = "Warranty created.",
        content = @Content(mediaType = APPLICATION_JSON_VALUE,
            examples = @ExampleObject(value = WarrantyResponse.EXAMPLE)))
    @ApiResponse(responseCode = "400", description = "Invalid request body.")
    @ApiResponse(responseCode = "404", description = "Warranty not found.")
    WarrantyResponse create(
        @RequestBody(content = @Content(examples =
            @ExampleObject(value = CreateWarrantyCommand.EXAMPLE)))
        CreateWarrantyCommand command);
}
```

### Service — business logic, transactional

- Concrete class: `@Service @Transactional @RequiredArgsConstructor @Slf4j`.
  Class-level `@Transactional`; `@Transactional(readOnly = true)` on pure reads.
- No `Authentication` param / `authentication.getName()` by default; fetch
  current user via the security-context utility only when truly required.
- Validation split: structural → Bean Validation on the request record (never
  re-check); business/cross-field/stateful → private `validate(command)` throwing
  the feature's `*BadRequest`.
- Map via injected MapStruct mapper; return response records, never entities.
- Updates via the mapper's `@MappingTarget` partial-update method.
- **No error logging** — `log.info` start/success with key ids, `log.warn`
  recoverable anomalies. Parameterized messages, never concatenation.

```java
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class WarrantyService {

    private final WarrantyRepository warrantyRepository;
    private final WarrantyMapper mapper;

    public WarrantyResponse create(CreateWarrantyCommand command) {
        log.info("Creating warranty '{}'", command.name());
        validate(command);                       // business rules only
        var saved = warrantyRepository.save(mapper.toEntity(command));
        log.info("Warranty {} created", saved.getId());
        return mapper.toResponse(saved);
    }

    @Transactional(readOnly = true)
    public WarrantyResponse getById(long id) {
        return warrantyRepository.findById(id)
                .map(mapper::toResponse)
                .orElseThrow(() -> new WarrantyNotFoundException(id));
    }

    public WarrantyResponse update(UpdateWarrantyCommand command) {
        var warranty = warrantyRepository.findById(command.warrantyId())
                .orElseThrow(() -> new WarrantyNotFoundException(command.warrantyId()));
        mapper.updateEntity(command, warranty);  // null props ignored
        return mapper.toResponse(warrantyRepository.save(warranty));
    }

    private void validate(CreateWarrantyCommand command) {
        if (command.startDate().isAfter(command.endDate())) {
            throw new WarrantyBadRequest("Start date can't be after end date.");
        }
    }
}
```

### Repository — Spring Data JPA

- `@Repository public interface XxxRepository extends JpaRepository<Entity, Id>`.
- Prefer derived query methods. Use `@EntityGraph(attributePaths = {...})`
  whenever you fetch an entity whose children you access afterward — solve N+1
  here, not via EAGER. Aggregates/complex reads: `@Query` JPQL + `@Param`; if
  native is unavoidable, never hardcode a schema; return a projection interface
  for multi-column aggregates.

```java
@Repository
public interface WarrantyRepository extends JpaRepository<Warranty, Long> {

    @EntityGraph(attributePaths = "files")
    List<Warranty> findAllByStatus(WarrantyStatus status);

    List<Warranty> findByEndDateBeforeAndStatusIn(LocalDate date,
                                                  List<WarrantyStatus> statuses);
}
```

### Entity — JPA + Lombok

- `@Entity @Getter @Setter @NoArgsConstructor @Accessors(chain = true)`,
  `@Table(name = "...")`, `@EntityListeners(AuditingEntityListener.class)`.
- `@Id @GeneratedValue(strategy = IDENTITY)` for app-generated ids; assigned
  `String` id for externally-owned ids.
- **UPPERCASE** columns: `@Column(name = "START_DATE")`, `@JoinColumn(name = "USER_ID")`.
  `@Enumerated(STRING)` (never ordinal).
- Associations `LAZY` always — `@ManyToOne(fetch = LAZY)`, never EAGER; fetch
  children via `@EntityGraph`/fetch joins.
- Owning collections: `@OneToMany(mappedBy=..., cascade = ALL, orphanRemoval = true)`,
  init to `new ArrayList<>()`.
- `equals`/`hashCode` id-based, `final`, `instanceof` pattern. No
  `@Data`/`@EqualsAndHashCode` on entities.
- Auditing automatic — never set timestamps in services. Enable once with a
  `@Configuration @EnableJpaAuditing` class under `config/`.

```java
@Entity
@Getter @Setter @NoArgsConstructor
@Accessors(chain = true)
@Table(name = "WARRANTIES")
@EntityListeners(AuditingEntityListener.class)
public class Warranty {

    @Id @GeneratedValue(strategy = IDENTITY)
    private long id;

    @Column(name = "NAME")
    private String name;

    @Column(name = "STATUS")
    @Enumerated(STRING)
    private WarrantyStatus status;

    @OneToMany(mappedBy = "warranty", cascade = ALL, orphanRemoval = true)
    private List<WarrantyFile> files = new ArrayList<>();

    @ManyToOne(fetch = LAZY)
    @JoinColumn(name = "USER_ID")
    private User user;

    @CreatedDate  @Column(name = "CREATED_AT", updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate @Column(name = "UPDATED_AT")
    private LocalDateTime updatedAt;

    @Override public final boolean equals(Object o) {
        return this == o || (o instanceof Warranty other && id == other.id);
    }
    @Override public final int hashCode() { return Objects.hashCode(id); }
}
```

### DTOs — records, validated, self-documenting

- Requests: `Create<Entity>Command` / `Update<Entity>Command`. Responses:
  `<Entity>Response` (never `...DTO`). Both are `record`s.
- Every component: `@Schema(description, example)`. Requests carry Bean
  Validation with user-facing `message`s.
- Every record declares `public static final String EXAMPLE` — a JSON text block
  honoring the record's own validations. Enum fields list every option as
  `"ACTIVE | EXPIRED | CLAIMED_ACTIVE | CLAIMED_EXPIRED | null"`. The Operations
  interface references these constants.

```java
public record CreateWarrantyCommand(

        @Schema(description = "Warranty display name", example = "Samsung TV warranty")
        @NotBlank(message = "Provide warranty name.")
        @Size(min = 2, max = 64, message = "Name length must be between 2 and 64.")
        String name,

        @Schema(description = "Free-text note", example = "Receipt in the drawer")
        String note,

        @Schema(description = "Coverage start date", example = "2026-01-01")
        @NotNull(message = "Provide start date.")
        LocalDate startDate,

        @Schema(description = "Coverage end date", example = "2028-01-01")
        @NotNull(message = "Provide end date.")
        LocalDate endDate,

        @Schema(description = "Optional category", example = "Electronics")
        @Size(min = 2, max = 64, message = "Category size must be between 2 and 64.")
        String category
) {
    public static final String EXAMPLE = """
        {
          "name": "Samsung TV warranty",
          "note": "Receipt in the drawer",
          "startDate": "2026-01-01",
          "endDate": "2028-01-01",
          "category": "Electronics"
        }
        """;
}

public record WarrantyResponse(

        @Schema(description = "Warranty id", example = "1")
        long id,

        @Schema(description = "Warranty display name", example = "Samsung TV warranty")
        String name,

        @Schema(description = "Coverage start date", example = "2026-01-01")
        LocalDate startDate,

        @Schema(description = "Coverage end date", example = "2028-01-01")
        LocalDate endDate,

        @Schema(description = "Lifecycle status",
                example = "ACTIVE | EXPIRED | CLAIMED_ACTIVE | CLAIMED_EXPIRED | null")
        WarrantyStatus status,

        @Schema(description = "Category", example = "Electronics")
        String category,

        Metadata metadata
) {
    public record Metadata(
            @Schema(description = "Note", example = "Receipt in the drawer")
            String note,
            @Schema(description = "Created timestamp", example = "2026-01-01T10:15:30")
            LocalDateTime createdAt,
            @Schema(description = "Updated timestamp", example = "2026-02-01T08:00:00")
            LocalDateTime updatedAt) {}

    public static final String EXAMPLE = """
        {
          "id": 1,
          "name": "Samsung TV warranty",
          "startDate": "2026-01-01",
          "endDate": "2028-01-01",
          "status": "ACTIVE | EXPIRED | CLAIMED_ACTIVE | CLAIMED_EXPIRED | null",
          "category": "Electronics",
          "metadata": {
            "note": "Receipt in the drawer",
            "createdAt": "2026-01-01T10:15:30",
            "updatedAt": "2026-02-01T08:00:00"
          }
        }
        """;
}
```

### Mapper — MapStruct

```java
@Mapper(componentModel = "spring",
        nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface WarrantyMapper {

    Warranty toEntity(CreateWarrantyCommand command);

    @Mapping(source = "note",      target = "metadata.note")
    @Mapping(source = "createdAt", target = "metadata.createdAt")
    @Mapping(source = "updatedAt", target = "metadata.updatedAt")
    WarrantyResponse toResponse(Warranty warranty);

    List<WarrantyResponse> toResponse(List<Warranty> warranties);

    @Mapping(target = "id", ignore = true)
    void updateEntity(UpdateWarrantyCommand command, @MappingTarget Warranty entity);
}
```

`NullValuePropertyMappingStrategy.IGNORE` replaces hand-written null ternaries.

### Exceptions — typed, centralized, logged only here

- Domain exceptions extend `CustomResponseStatusException`, pass an `ErrorCode`
  (add an `ERR0xx` constant + message per new category). One class per condition
  in `exception/custom/`: `XxxNotFoundException` (404), `XxxBadRequest` (400),
  etc. Never throw raw `RuntimeException`/`ResponseStatusException`.
- `GlobalExceptionHandler` (`@RestControllerAdvice`) is the **only** place that
  logs errors (`log.error`) and builds the body. Responses are typed
  `ProblemDetail`/`ApiError`, never `Map`/`HashMap`. The generic `Exception`
  handler returns a safe generic message and logs the real cause. Keep dedicated
  handlers for `MethodArgumentNotValidException` (field errors → 400) and
  `DataIntegrityViolationException` (409).

```java
@ExceptionHandler(CustomResponseStatusException.class)
ProblemDetail handle(CustomResponseStatusException ex) {
    log.error("Handled domain exception [{}]", ex.getErrorCode(), ex);
    var pd = ProblemDetail.forStatusAndDetail(ex.getHttpStatus(), ex.getMessage());
    pd.setProperty("errorCode", ex.getErrorCode());
    return pd;
}

@ExceptionHandler(Exception.class)
ProblemDetail handle(Exception ex) {
    log.error("Unhandled exception", ex);
    return ProblemDetail.forStatusAndDetail(INTERNAL_SERVER_ERROR,
            "Unexpected error occurred.");
}
```

### Security — user-defined, JWT by default

- Mechanism is chosen by the user; **do not assume Keycloak**. Default JWT.
- For JWT, implement a `OncePerRequestFilter` that validates the token and
  populates `SecurityContext`; register it in a stateless `SecurityFilterChain`
  (CSRF disabled, `auth/**` + swagger public, all else authenticated).
- Controllers/services do not thread `Authentication`; when the current user is
  required, expose a small `SecurityContextHolder`-backed utility and call it
  only there.
- CORS: explicit allowed origins bound from properties — never `allowedOrigins("*")`.

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String token = resolveToken(request);
        if (token != null && jwtService.isValid(token)) {
            SecurityContextHolder.getContext()
                    .setAuthentication(jwtService.toAuthentication(token));
        }
        chain.doFilter(request, response);
    }
}
```

## Common cases

### Scheduled tasks — ShedLock

- Lock every scheduled method with ShedLock so only one instance runs it:
  `@EnableScheduling @EnableSchedulerLock(defaultLockAtMostFor = "...")` once;
  `@SchedulerLock(name = "...")` per method. `enabled` and `cron-expression`
  come from properties; gate execution on `enabled`.

```yaml
scheduling:
  tasks:
    synchronize-liability-insurance-task:
      name: synchronize-liability-insurance-task
      enabled: false
      cron-expression: 0 0 1 * * ?
    synchronize-comprehensive-insurance-task:
      name: synchronize-comprehensive-insurance-task
      enabled: false
      cron-expression: 0 0 2 * * ?
```

```java
public record SchedulingProperties(Map<String, TaskProperties> tasks) {
    public record TaskProperties(String name, boolean enabled, String cronExpression) {}
}

@Component
@RequiredArgsConstructor
@Slf4j
public class SynchronizeLiabilityInsuranceTask {

    private final ComplianceProperties properties;

    @Scheduled(cron = "#{@complianceProperties.scheduling().tasks()['synchronize-liability-insurance-task'].cronExpression()}")
    @SchedulerLock(name = "synchronize-liability-insurance-task")
    public void run() {
        var cfg = properties.scheduling().tasks().get("synchronize-liability-insurance-task");
        if (!cfg.enabled()) return;
        log.info("Running {}", cfg.name());
        // ... work
    }
}
```

### Configuration properties — nested records

All config binds to Java `record`s: one base `@ConfigurationProperties
<Application>Properties` record composing child records (which may nest further).
Keep `@ConfigurationPropertiesScan` on the application class. No magic
strings/URLs in code.

```java
public record BaseURLs(
        String liabilityInsurance,
        String technicalReview,
        String vignette
) {
    public static String buildURL(String baseUrl, String path) {
        return baseUrl + path;
    }
}

public record EndpointURLs(
        String vignetteProducts
) {}

@ConfigurationProperties
public record ComplianceProperties(
        BaseURLs baseURLs,
        EndpointURLs endpointURLs,
        SchedulingProperties scheduling
) {}
```

```yaml
baseURLs:
  liabilityInsurance: http://someBaseUrl:8080/liability-insurance/v1.0.0
  technicalReview: http://someBaseUrl:8080/technical-review/v1.0.0
  vignette: http://someBaseUrl:8080/vignette/v1.0.0

endpointURLs:
  vignetteProducts: /products
```

### Outbound API calls — RestClient

- `RestClient` only — never `RestTemplate`/other clients. Define a configured
  `RestClient` bean under `config/`. Build URLs with
  `BaseURLs.buildURL(rootUrl, endpointUrl)` from properties. Clients live in
  `client/`.

```java
@Component
@RequiredArgsConstructor
public class VignetteClient {

    private final RestClient restClient;
    private final ComplianceProperties properties;

    public List<VignetteProductResponse> getProducts() {
        String url = BaseURLs.buildURL(
                properties.baseURLs().vignette(),
                properties.endpointURLs().vignetteProducts());
        return restClient.get()
                .uri(url)
                .retrieve()
                .body(new ParameterizedTypeReference<>() {});
    }
}
```

### pom.xml

A comment above each dependency, or each coherent group, stating its purpose.

```xml
<!-- Web layer: REST controllers, embedded server -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Distributed lock for @Scheduled tasks (single-runner across instances) -->
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
</dependency>
```

## Feature checklist (the workflow)

Follow in order. Each step's rules are in the section named in brackets.

1. **Liquibase changelog** for the schema change — never `ddl-auto`. [Tech baseline]
2. **Entity** in `<feature>/model`: LAZY associations, JPA auditing, UPPERCASE
   columns, id/equals rules. [Entity]
3. **Repository** with `@EntityGraph` where children are accessed later. [Repository]
4. **Request records** (`Create/Update<Entity>Command`) + **response record**
   (`<Entity>Response`): `@Schema` on every component, `EXAMPLE` text block, Bean
   Validation on requests. [DTOs]
5. **Mapper**: `toEntity`, `toResponse`, list overload, `updateEntity`
   (`@MappingTarget`, IGNORE). [Mapper]
6. **Service**: `@Service @Transactional @Slf4j`; business-rule `validate()`;
   mapper for conversion/partial update; info/warn logging only. [Service]
7. **Operations interface**: full Swagger annotations referencing the records'
   `EXAMPLE` constants. [Operations interface]
8. **Controller** implementing it: thin, `@Valid`, direct-record return,
   static-imported `@ResponseStatus`, `/api/v1/...`, no `Authentication` param. [Controller]
9. **Exceptions**: add `ErrorCode` constant + `custom/` classes; confirm
   `GlobalExceptionHandler` covers them (typed `ProblemDetail`, error logged only
   there). [Exceptions]
10. If config/scheduling/outbound calls are involved: nested
    `@ConfigurationProperties` records, ShedLock-locked tasks, or a
    `RestClient`-based client per [Common cases].
11. Build to verify codegen: `./mvnw -q -DskipTests compile` (Lombok + MapStruct
    annotation processing must succeed).

## Output discipline

- Match existing formatting/import style; Jakarta imports only.
- No cross-layer leakage: entities never cross the API boundary; no HTTP
  concerns in services; services never log errors.
- You do not create tests or testing scaffolding under any circumstances.
