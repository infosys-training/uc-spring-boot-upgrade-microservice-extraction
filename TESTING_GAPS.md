# Testing Gaps: Spring Boot 3.2.5 Upgrade

## Test Scenarios Not Verifiable in This Environment

### 1. Selenium E2E Tests
- **Scenario:** Full browser-based end-to-end tests (login, article CRUD, comment flows, profile management)
- **Why not verified:** Selenium tests require a running Chrome browser with WebDriver and the full application stack (backend + frontend). These tests are excluded from the standard `./gradlew test` task and run via `./gradlew seleniumTest` with TestNG. The test environment does not have a browser or the Next.js frontend running.
- **Risk:** WebDriver API changes between Selenium versions may cause runtime failures. The `WebDriverWait(driver, Duration)` constructor change was fixed but other API differences may exist.

### 2. Integration Tests with External Services
- **Scenario:** Tests verifying behavior against real databases, message queues, or third-party APIs
- **Why not verified:** All tests use in-memory SQLite (`:memory:`) via the `test` profile. No integration tests exist that connect to external MySQL/PostgreSQL databases or external services.
- **Risk:** SQLite dialect differences vs production database; connection pooling behavior changes in HikariCP version bundled with Spring Boot 3.2.x.

### 3. GraphQL API Functional Tests
- **Scenario:** Full request/response cycle testing of GraphQL queries and mutations (article feed pagination, user creation via GraphQL, comment CRUD)
- **Why not verified:** No dedicated GraphQL integration tests exist in the test suite. The GraphQL layer (DGS data fetchers) is not tested via HTTP. Only REST API controllers have MockMvc-based tests.
- **Risk:** DGS 8.x behavioral changes in data fetcher resolution, exception handling, or connection/pagination semantics. The `handleException` (previously `onException`) method signature change was addressed but runtime behavior with Spring GraphQL integration may differ.

### 4. Security Regression Tests
- **Scenario:** Comprehensive authentication/authorization testing — JWT token validation edge cases, expired token handling, CORS preflight requests, endpoint access control
- **Why not verified:** Existing tests cover basic auth flows, but Spring Security 6's stricter `requestMatchers()` behavior vs the old `antMatchers()` has edge cases not covered. The switch from `WebSecurityConfigurerAdapter` to `SecurityFilterChain` bean may affect filter ordering.
- **Risk:** Subtle authorization bypass or over-restriction; CORS handling differences; Spring Security's new default behavior changes (e.g., CSRF, session management defaults).

### 5. Performance / Load Tests
- **Scenario:** Throughput, latency, and resource usage benchmarks comparing before/after upgrade
- **Why not verified:** No performance test suite exists. Spring Boot 3.x uses Tomcat 10.x (Jakarta Servlet) which has different performance characteristics than Tomcat 9.x (javax Servlet).
- **Risk:** Memory footprint changes, startup time differences, thread pool behavior changes.

### 6. Frontend Integration Tests
- **Scenario:** Next.js frontend communicating with the upgraded Spring Boot backend
- **Why not verified:** The frontend is a separate Next.js application in `frontend/`. It was not built or tested as part of this upgrade. API contract changes (if any) would break the frontend.
- **Risk:** REST API response format changes; Jackson serialization differences between Spring Boot 2.x and 3.x; CORS behavior changes.

### 7. Database Migration Tests
- **Scenario:** Flyway migration compatibility with production databases; migration from existing schema to upgraded schema
- **Why not verified:** Tests use `spring.flyway.target=1` (only V1 migration). Production may have additional migrations or data that could conflict.
- **Risk:** Flyway version bundled with Spring Boot 3.2.x may handle SQLite migrations differently; schema validation behavior changes.

### 8. Security Scanning
- **Scenario:** OWASP dependency check, CVE scanning of upgraded dependencies
- **Why not verified:** No NVD API key configured; `dependencyCheckAnalyze` task is not in the build pipeline.
- **Risk:** Upgraded dependencies may introduce new CVEs not present in the original versions.

### 9. Docker / Container Tests
- **Scenario:** Building and running the application in Docker containers
- **Why not verified:** No Dockerfile exists in the repository. Container-based deployment was not tested.
- **Risk:** Java 17 base image requirements; container startup behavior changes.

### 10. Actuator / Health Check Tests
- **Scenario:** Spring Boot Actuator endpoints (if enabled), readiness/liveness probes
- **Why not verified:** No Actuator dependency is included. If Actuator is added later, Spring Boot 3.x has changed default endpoint exposure rules.
- **Risk:** N/A currently, but relevant if Actuator is added.

## Recommended Follow-up Actions

1. **Run Selenium E2E suite** (`./gradlew seleniumTest`) in a CI environment with Chrome/ChromeDriver
2. **Test with production database** (MySQL/PostgreSQL) to verify Flyway migrations and query compatibility
3. **Add GraphQL integration tests** to cover DGS 8.x data fetcher behavior
4. **Run OWASP dependency check** with NVD API key to verify no new CVEs
5. **Performance benchmark** comparing startup time and request latency before/after upgrade
6. **Frontend smoke test** to verify API contract compatibility with the Next.js app
