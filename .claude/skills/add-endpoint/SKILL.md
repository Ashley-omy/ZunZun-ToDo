---
name: add-endpoint
description: Add a new REST API endpoint to the ZunZun ToDo backend, producing the controller, service, repository, DTOs, and mapper according to project conventions. Use when the user says "add an API", "add an endpoint", "implement CRUD", "build the backend for X", or whenever starting server-side work for a feature.
---

# Adding a REST Endpoint

Check the endpoint's contract in `docs/design-specification.md` §7 first. If the implementation must differ,
update the SDS in the same pull request.

## Layering

```
Controller  … HTTP in/out only; no business logic
   ↓ DTO (record)
Service     … business rules, authorization, transaction boundary
   ↓ Entity
Repository  … data access only
```

**Entities never leave the service layer.**

## Where Code Goes

```
com.zunzuntodo.{feature}/
├── XxxController.java
├── XxxService.java
├── XxxRepository.java
├── Xxx.java              (entity)
├── XxxMapper.java        (MapStruct)
└── dto/
    ├── CreateXxxRequest.java
    └── XxxResponse.java
```

## API Conventions

- Base path `/api/v1`; plural resource names; `PATCH` for partial updates
- Pagination `?page=&size=` (the feed uses a cursor)
- Authenticated by default; public endpoints are allow-listed explicitly in `SecurityConfig`
- The current user comes from the JWT: `@AuthenticationPrincipal Jwt jwt` → `Long.valueOf(jwt.getSubject())`

| Situation | Status |
|---|---|
| Read / update succeeded | 200 |
| Created | 201 + `Location` |
| Deleted | 204 |
| Malformed request / validation failed | 400 |
| Valid request breaking a business rule (e.g. future date) | 422 |
| Not authenticated | 401 |
| Not allowed on a resource the caller can see | 403 |
| Not found — **or another user's private resource** | 404 |
| Conflict (e.g. second completion of a task, handle taken) | 409 |

## Requirements

**Authorization (in the service, never the controller)**
- Owned resources: load by id and owner, e.g. `findByIdAndOwnerId`; if absent → `ResourceNotFoundException` (404)
- Other users' data: only `PUBLIC` projects; achievements only when `PUBLIC` and `COMPLETED`
- Progress records: owner only, always

**Validation**
- Annotations on request DTOs (`@NotBlank`, `@Size`, `@Min`, `@Max`, `@Pattern`); `@Valid` on the controller parameter
- Business rules in the service, throwing domain exceptions (`BusinessRuleException` → 422, `ConflictException` → 409)

**Errors**
- `GlobalExceptionHandler` maps exceptions to RFC 7807. Never try/catch in controllers
- Map `DataIntegrityViolationException` from known unique constraints to 409

**Transactions**
- `@Transactional` on methods writing more than once; `@Transactional(readOnly = true)` on reads

**Time**
- Inject `Clock`; compute "today" as `LocalDate.now(clock.withZone(ZoneId.of(user.getTimeZone())))`
- Calendar dates are `LocalDate`; moments are `Instant`; durations are `int` minutes

**Progress records**
- Anything that completes or un-completes a task goes through `ProgressLogService`, which keeps the task status and
  the completion record consistent in one transaction

**OpenAPI**
- `@Tag` on the controller, `@Operation(summary = "...")` on each method

## Templates

```java
@RestController
@RequestMapping("/api/v1/projects")
@Tag(name = "Projects")
public class ProjectController {

    private final ProjectService projectService;

    public ProjectController(ProjectService projectService) {
        this.projectService = projectService;
    }

    @PostMapping
    @Operation(summary = "Create a project")
    public ResponseEntity<ProjectResponse> create(@AuthenticationPrincipal Jwt jwt,
                                                  @Valid @RequestBody CreateProjectRequest request) {
        ProjectResponse created = projectService.create(userId(jwt), request);
        return ResponseEntity.created(URI.create("/api/v1/projects/" + created.id())).body(created);
    }

    @GetMapping("/{id}")
    @Operation(summary = "Get one of my projects")
    public ProjectResponse get(@AuthenticationPrincipal Jwt jwt, @PathVariable Long id) {
        return projectService.getOwned(userId(jwt), id);
    }

    private static Long userId(Jwt jwt) {
        return Long.valueOf(jwt.getSubject());
    }
}
```

```java
@Service
public class ProjectService {

    private final ProjectRepository projectRepository;
    private final ProjectMapper projectMapper;

    public ProjectService(ProjectRepository projectRepository, ProjectMapper projectMapper) {
        this.projectRepository = projectRepository;
        this.projectMapper = projectMapper;
    }

    @Transactional
    public ProjectResponse create(Long userId, CreateProjectRequest request) {
        Project project = projectMapper.toEntity(request);
        project.setOwnerId(userId);
        return projectMapper.toResponse(projectRepository.save(project));
    }

    @Transactional(readOnly = true)
    public ProjectResponse getOwned(Long userId, Long projectId) {
        return projectRepository.findByIdAndOwnerId(projectId, userId)
                .map(projectMapper::toResponse)
                .orElseThrow(() -> new ResourceNotFoundException("Project", projectId));
    }
}
```

```java
public record CreateProjectRequest(
        @NotBlank @Size(max = 120) String title,
        @Size(max = 2000) String description,
        LocalDate targetDate,
        @NotNull Visibility visibility
) {}
```

## Checklist

- [ ] Matches SDS §7 (or the SDS was updated)
- [ ] Controller, service, repository, DTOs, mapper present; no entity in the controller layer
- [ ] Ownership/visibility enforced in the service; other users' private data returns 404
- [ ] Validation on DTOs; business rules in the service with the right status codes
- [ ] `@Transactional` where needed; `Clock` used for "today"
- [ ] OpenAPI annotations
- [ ] Tests written (write-tests skill); `./mvnw test` passes
