---
name: write-tests
description: Write backend tests for ZunZun ToDo — JUnit Jupiter unit tests and Testcontainers integration tests. Use when the user says "write tests", "add a test", or "improve coverage", and immediately after implementing any new service or controller. Covers the required test areas — authorization and visibility, task completion consistency, chart date ranges, and authentication.
---

# Writing Tests

## Two Kinds

| Kind | File name | Covers | Real database |
|---|---|---|---|
| Unit test | `XxxTest.java` | Service rules, calculations | No (Mockito) |
| Integration test | `XxxIT.java` | Full HTTP API, repositories, migrations | Yes (Testcontainers) |

Integration tests extend `AbstractIntegrationTest`, which starts one PostgreSQL container for the whole run.
Spring Boot 4's `@ServiceConnection` wires the datasource automatically.
Testcontainers 2 moved the class to `org.testcontainers.postgresql.PostgreSQLContainer` and dropped the type parameter:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
public abstract class AbstractIntegrationTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer postgres = new PostgreSQLContainer("postgres:17-alpine");
}
```

Use `@MockitoBean` (not the removed `@MockBean`).

Maven Surefire runs `*Test`, Failsafe runs `*IT` (`./mvnw verify` runs both; `./mvnw test` runs unit tests).
**Never use H2.** Partial indexes and check constraints behave differently there.

Use a fixed `Clock` bean in tests (`Clock.fixed(...)`) so "today" is deterministic.

## Naming

Method names read as a sentence: `returns404WhenReadingAnotherUsersPrivateProject()`.
Structure bodies with `// given`, `// when`, `// then`.

---

## Required Test Areas

### 1. Authorization and Visibility (highest priority)

For every endpoint that touches another user's data, prove that private data never leaks:

```java
@Test void returns404WhenReadingAnotherUsersPrivateProject() { }
@Test void privateProjectsAreExcludedFromPublicProfile() { }
@Test void completedPrivateProjectDoesNotAppearInFollowersFeed() { }
@Test void reopenedProjectDisappearsFromFeedAndRejectsNewComments() { }
@Test void progressRecordsAreNeverReturnedForAnotherUser() { }
@Test void cannotEditAnotherUsersComment() { }               // 403
@Test void achievementOwnerCanDeleteAnyCommentOnIt() { }
```

### 2. Task Completion Consistency

Progress records are the single source of truth for completion.

```java
@Test void recordingACompletionMarksTheTaskDone() { }
@Test void deletingTheCompletionRecordMarksTheTaskTodo() { }
@Test void secondCompletionForTheSameTaskReturns409() { }
@Test void completingFromTheProjectPageCreatesARecordForToday() { }
@Test void recordsCannotBeDatedInTheUsersFuture() { }        // 422, uses the user's time zone
@Test void minutesOutsideOneTo1440AreRejected() { }
```

### 3. Chart Data

```java
@Test void weekRunsMondayToSundayAroundTheGivenDate() { }
@Test void monthHasOnePointPerDayIncludingFebruaryInALeapYear() { }
@Test void daysWithoutRecordsAreReturnedAsZero() { }
@Test void completedTasksMetricCountsOnlyCompletionRecords() { }
@Test void projectScopeOnlyCountsThatProject() { }
@Test void recordsWithoutMinutesAddNothingToTheTimeMetric() { }
```

### 4. Authentication

```java
@Test void refreshRotatesTheTokenAndInvalidatesTheOldOne() { }
@Test void reusingARotatedRefreshTokenRevokesTheWholeFamily() { }
@Test void refreshIsRejectedFromAnUnknownOrigin() { }
@Test void googleSignInIsRefusedForAnEmailWithAPasswordAccount() { }
@Test void loginIsRateLimitedPerIp() { }
@Test void protectedEndpointsReturn401WithoutAToken() { }
```

### 5. Uploads and Account Deletion

```java
@Test void avatarUploadRejectsNonImagesEvenWithAnImageExtension() { }
@Test void avatarUploadRejectsFilesOver2Mb() { }
@Test void deletingTheAccountRemovesAllDataAndTheProfileImage() { }
```

---

## Test Data

- Small factory helpers (`givenUser()`, `givenProject(owner, PUBLIC)`) — each test overrides only what it needs
- Obtain tokens through the real `/auth/register` or a test helper that signs a JWT with the test key
- Keep tests independent: clean tables between integration tests (truncate helper) rather than relying on order

## Coverage Policy

Aim for 70%+ on service-layer logic. The areas above matter far more than the percentage;
don't chase coverage on getters or mappers.

## Checklist

- [ ] Failure paths covered (400 / 401 / 403 / 404 / 409 / 422), not only the happy path
- [ ] Every path returning another user's data has a visibility test
- [ ] Completion consistency and chart date ranges tested
- [ ] Fixed `Clock` used wherever "today" matters
- [ ] Integration tests use real PostgreSQL
- [ ] `./mvnw verify` is green
