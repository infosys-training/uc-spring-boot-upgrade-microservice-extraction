# Migration Proof: Spring Boot 2.6.3 → 3.2.5 + Java 11 → 17

## Before State

| Component              | Version Before |
|------------------------|---------------|
| Java (source/target)   | 11            |
| Spring Boot            | 2.6.3         |
| Spring Dependency Mgmt | 1.0.11.RELEASE|
| Gradle                 | 7.4           |
| Gradle Wrapper         | 7.4-bin       |
| MyBatis Spring Boot    | 2.2.2         |
| DGS Framework          | 4.9.21        |
| DGS Codegen Plugin     | 5.0.6         |
| JJWT                   | 0.11.2        |
| Flyway Core            | (managed)     |
| SQLite JDBC            | 3.36.0.3      |
| Joda-Time              | 2.10.13       |
| Rest-Assured           | 4.5.1         |
| Mockito Inline         | 4.0.0         |
| Spotless               | 6.2.1         |
| JaCoCo                 | 0.8.7         |
| Selenium               | 4.15.0        |
| Lombok                 | (managed)     |

**Build result:** BUILD SUCCESSFUL — 68 tests, 0 failures  
**Compilation warnings:** Deprecated API usage in GraphQLCustomizeExceptionHandler

## After State

| Component              | Version After  |
|------------------------|---------------|
| Java (source/target)   | 17            |
| Spring Boot            | 3.2.5         |
| Spring Dependency Mgmt | 1.1.4         |
| Gradle                 | 8.5           |
| Gradle Wrapper         | 8.5-bin       |
| MyBatis Spring Boot    | 3.0.3         |
| DGS Framework          | 8.7.1 (spring-graphql-starter) |
| DGS Codegen Plugin     | 6.2.1         |
| JJWT                   | 0.12.5        |
| Flyway Core            | (managed)     |
| SQLite JDBC            | 3.45.1.0      |
| Joda-Time              | 2.12.7        |
| Rest-Assured           | 5.4.0         |
| Mockito Inline         | removed (managed by Boot) |
| Spotless               | 6.25.0        |
| JaCoCo                 | 0.8.11        |
| Selenium               | 4.15.0        |
| Lombok                 | (managed)     |

**Build result:** BUILD SUCCESSFUL — 68 tests, 0 failures  
**Compilation errors:** 0  
**Remaining javax.* imports in src/:** 0 (only JDK javax.crypto in DefaultJwtService)

## Diff Summary

### build.gradle
- Spring Boot plugin: `2.6.3` → `3.2.5`
- Dependency management plugin: `1.0.11.RELEASE` → `1.1.4`
- DGS Codegen plugin: `5.0.6` → `6.2.1`
- Spotless plugin: `6.2.1` → `6.25.0`
- `sourceCompatibility` / `targetCompatibility`: `'11'` → `'17'`
- MyBatis starter: `2.2.2` → `3.0.3`
- DGS starter: `graphql-dgs-spring-boot-starter:4.9.21` → `graphql-dgs-spring-graphql-starter:8.7.1`
- JJWT: `0.11.2` → `0.12.5`
- SQLite JDBC: `3.36.0.3` → `3.45.1.0`
- Joda-Time: `2.10.13` → `2.12.7`
- Rest-Assured: `4.5.1` → `5.4.0`
- JaCoCo toolVersion: `0.8.7` → `0.8.11`
- Removed explicit `mockito-inline` (managed by Spring Boot 3.x)
- Spotless target changed from project root to `src/**/*.java` to avoid Gradle 8.x implicit dependency issues

### gradle/wrapper/gradle-wrapper.properties
- Gradle distribution: `7.4-bin` → `8.5-bin`

### javax → jakarta Namespace Migration (20 files)
All `javax.validation.*` → `jakarta.validation.*`  
All `javax.servlet.*` → `jakarta.servlet.*`  
`javax.crypto.*` retained (JDK package, not Jakarta EE)

**Files migrated:**
- `src/main/java/io/spring/api/ArticleApi.java`
- `src/main/java/io/spring/api/ArticlesApi.java`
- `src/main/java/io/spring/api/CommentsApi.java`
- `src/main/java/io/spring/api/CurrentUserApi.java`
- `src/main/java/io/spring/api/UsersApi.java`
- `src/main/java/io/spring/api/exception/CustomizeExceptionHandler.java`
- `src/main/java/io/spring/api/security/JwtTokenFilter.java`
- `src/main/java/io/spring/application/article/ArticleCommandService.java`
- `src/main/java/io/spring/application/article/DuplicatedArticleConstraint.java`
- `src/main/java/io/spring/application/article/DuplicatedArticleValidator.java`
- `src/main/java/io/spring/application/article/NewArticleParam.java`
- `src/main/java/io/spring/application/user/DuplicatedEmailConstraint.java`
- `src/main/java/io/spring/application/user/DuplicatedEmailValidator.java`
- `src/main/java/io/spring/application/user/DuplicatedUsernameConstraint.java`
- `src/main/java/io/spring/application/user/DuplicatedUsernameValidator.java`
- `src/main/java/io/spring/application/user/RegisterParam.java`
- `src/main/java/io/spring/application/user/UpdateUserParam.java`
- `src/main/java/io/spring/application/user/UserService.java`
- `src/main/java/io/spring/graphql/UserMutation.java`
- `src/main/java/io/spring/graphql/exception/GraphQLCustomizeExceptionHandler.java`

### Spring Security 6 Migration
**`WebSecurityConfig.java`:**
- Removed `WebSecurityConfigurerAdapter` (deleted in Spring Security 6)
- Replaced `configure(HttpSecurity)` override with `@Bean SecurityFilterChain`
- Migrated from method-chaining DSL to lambda DSL
- `antMatchers()` → `requestMatchers()`
- `authorizeRequests()` → `authorizeHttpRequests()`
- `csrf().disable()` → `csrf(csrf -> csrf.disable())`

### JJWT 0.12.x Migration
**`DefaultJwtService.java`:**
- Removed `SignatureAlgorithm` enum usage (deprecated)
- `setSubject()` → `subject()`
- `setExpiration()` → `expiration()`
- `Jwts.parserBuilder().setSigningKey()` → `Jwts.parser().verifyWith()`
- `parseClaimsJws()` → `parseSignedClaims()`
- `claimsJws.getBody()` → `claimsJws.getPayload()`

### DGS 8.x / GraphQL Java 21.x Migration
**`GraphQLCustomizeExceptionHandler.java`:**
- `onException()` → `handleException()` returning `CompletableFuture<DataFetcherExceptionHandlerResult>`

**`ArticleDatafetcher.java` / `CommentDatafetcher.java`:**
- Replaced `graphql.relay.DefaultPageInfo` / `DefaultConnectionCursor` with DGS-generated `io.spring.graphql.types.PageInfo` builder

### Spring MVC 6 Migration
**`CustomizeExceptionHandler.java`:**
- `handleMethodArgumentNotValid` parameter type: `HttpStatus` → `HttpStatusCode`

### Selenium WebDriverWait Fix
**`BasePage.java`:**
- `new WebDriverWait(driver, long)` → `new WebDriverWait(driver, Duration.ofSeconds(long))`

### Application Properties
- Added `spring.graphql.schema.locations=classpath:schema/`
- Added `spring.graphql.schema.inspection.enabled=false`

## Test Results

| Metric | Before | After |
|--------|--------|-------|
| Total tests | 68 | 68 |
| Passed | 68 | 68 |
| Failed | 0 | 0 |
| Skipped | 0 | 0 |
| Compilation errors | 0 | 0 |
| javax.* imports in src/ | 40 | 0 |
