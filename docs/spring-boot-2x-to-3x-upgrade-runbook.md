# Spring Boot 2.x → 3.x Upgrade Runbook

A repeatable, step-by-step guide for upgrading Spring Boot 2.x services to Spring Boot 3.2 with Java 17. Derived from a production migration of a RealWorld blogging backend (REST + GraphQL, MyBatis, JWT auth, Flyway, SQLite).

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Upgrade Gradle & Java](#2-upgrade-gradle--java)
3. [Upgrade Spring Boot & Dependency Management](#3-upgrade-spring-boot--dependency-management)
4. [javax → jakarta Namespace Migration](#4-javax--jakarta-namespace-migration)
5. [Spring Security 6 Migration](#5-spring-security-6-migration)
6. [JJWT Library Migration](#6-jjwt-library-migration)
7. [DGS / GraphQL Migration](#7-dgs--graphql-migration)
8. [Spring MVC / Web Changes](#8-spring-mvc--web-changes)
9. [Test Dependencies & Fixes](#9-test-dependencies--fixes)
10. [Other Common Dependency Upgrades](#10-other-common-dependency-upgrades)
11. [Verification Checklist](#11-verification-checklist)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Prerequisites

- **Java 17** installed (Spring Boot 3.x requires Java 17+)
- **Gradle 8.x** (or Maven 3.8+)
- All tests passing on the current Spring Boot 2.x baseline
- A feature branch for the upgrade

```bash
git checkout -b upgrade/spring-boot-3.2
```

---

## 2. Upgrade Gradle & Java

### Gradle wrapper

```bash
./gradlew wrapper --gradle-version 8.5
```

### build.gradle — Java version

```groovy
// Before
sourceCompatibility = '11'
targetCompatibility = '11'

// After
sourceCompatibility = '17'
targetCompatibility = '17'
```

### Gradle properties (if using Spotless with Google Java Format)

Spotless's Google Java Format requires JVM `--add-exports` flags. Create or update `gradle.properties`:

```properties
org.gradle.jvmargs=--add-exports jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED \
  --add-exports jdk.compiler/com.sun.tools.javac.file=ALL-UNNAMED \
  --add-exports jdk.compiler/com.sun.tools.javac.parser=ALL-UNNAMED \
  --add-exports jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED \
  --add-exports jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED
```

---

## 3. Upgrade Spring Boot & Dependency Management

```groovy
plugins {
    id 'org.springframework.boot' version '3.2.5'        // was 2.6.x
    id 'io.spring.dependency-management' version '1.1.4' // was 1.0.x
}
```

---

## 4. javax → jakarta Namespace Migration

Spring Boot 3.x uses Jakarta EE 9+. All `javax.*` Jakarta EE imports must change to `jakarta.*`.

### Packages that MUST change

| Old (javax)               | New (jakarta)               |
|---------------------------|-----------------------------|
| `javax.validation.*`      | `jakarta.validation.*`      |
| `javax.servlet.*`         | `jakarta.servlet.*`         |
| `javax.persistence.*`     | `jakarta.persistence.*`     |
| `javax.annotation.*`      | `jakarta.annotation.*`      |
| `javax.transaction.*`     | `jakarta.transaction.*`     |
| `javax.inject.*`          | `jakarta.inject.*`          |
| `javax.ws.rs.*`           | `jakarta.ws.rs.*`           |

### Packages that stay as javax (JDK packages)

- `javax.crypto.*` — JDK cryptography
- `javax.net.*` — JDK networking
- `javax.sql.*` — JDK SQL
- `javax.swing.*` — JDK Swing
- `javax.xml.*` — JDK XML

### Batch migration command

```bash
# Validation
find src -name "*.java" -exec sed -i 's/import javax\.validation\./import jakarta.validation./g' {} +

# Servlet
find src -name "*.java" -exec sed -i 's/import javax\.servlet\./import jakarta.servlet./g' {} +

# Persistence (if using JPA)
find src -name "*.java" -exec sed -i 's/import javax\.persistence\./import jakarta.persistence./g' {} +

# Annotation
find src -name "*.java" -exec sed -i 's/import javax\.annotation\./import jakarta.annotation./g' {} +
```

### Verify no Jakarta EE javax imports remain

```bash
grep -rn "import javax\." src/ --include="*.java" | grep -v "javax\.crypto\." | grep -v "javax\.net\." | grep -v "javax\.sql\." | grep -v "javax\.swing\." | grep -v "javax\.xml\."
# Should return empty
```

---

## 5. Spring Security 6 Migration

### 5a. Remove WebSecurityConfigurerAdapter

`WebSecurityConfigurerAdapter` is removed in Spring Security 6. Replace with a `@Bean SecurityFilterChain`.

```java
// BEFORE (Spring Security 5.x)
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .cors()
            .and()
            .authorizeRequests()
            .antMatchers(HttpMethod.GET, "/api/**").permitAll()
            .anyRequest().authenticated();
    }
}

// AFTER (Spring Security 6.x)
@Configuration
@EnableWebSecurity
public class WebSecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.csrf(AbstractHttpConfigurer::disable)
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.GET, "/api/**").permitAll()
                .anyRequest().authenticated());
        return http.build();
    }
}
```

### 5b. API renames

| Spring Security 5              | Spring Security 6                |
|---------------------------------|----------------------------------|
| `.authorizeRequests()`          | `.authorizeHttpRequests()`       |
| `.antMatchers(...)`             | `.requestMatchers(...)`          |
| `.csrf().disable()`            | `.csrf(AbstractHttpConfigurer::disable)` |
| `.cors().and()`                | `.cors(cors -> cors.configurationSource(...))` |
| `.sessionManagement().sessionCreationPolicy(...)` | `.sessionManagement(s -> s.sessionCreationPolicy(...))` |
| `.exceptionHandling().authenticationEntryPoint(...)` | `.exceptionHandling(e -> e.authenticationEntryPoint(...))` |

### 5c. Required imports

```java
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.web.SecurityFilterChain;
```

---

## 6. JJWT Library Migration

### Version upgrade

```groovy
// Before
implementation 'io.jsonwebtoken:jjwt-api:0.11.2'
runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.11.2', 'io.jsonwebtoken:jjwt-jackson:0.11.2'

// After
implementation 'io.jsonwebtoken:jjwt-api:0.12.5'
runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.5', 'io.jsonwebtoken:jjwt-jackson:0.12.5'
```

### API changes

```java
// BEFORE (0.11.x)
// Key construction
signatureAlgorithm = SignatureAlgorithm.HS512;
signingKey = new SecretKeySpec(secret.getBytes(), signatureAlgorithm.getJcaName());

// Token creation
Jwts.builder()
    .setSubject(user.getId())
    .setExpiration(expireTime)
    .signWith(signingKey)
    .compact();

// Token parsing
Jwts.parserBuilder().setSigningKey(signingKey).build()
    .parseClaimsJws(token).getBody().getSubject();

// AFTER (0.12.x)
// Key construction — use Keys.hmacShaKeyFor with >= 64 bytes
byte[] keyBytes = secret.getBytes(StandardCharsets.UTF_8);
if (keyBytes.length < 64) {
    throw new IllegalArgumentException(
        "JWT secret must be at least 64 bytes for HS512. Current length: " + keyBytes.length);
}
signingKey = Keys.hmacShaKeyFor(keyBytes);

// Token creation
Jwts.builder()
    .subject(user.getId())        // was setSubject()
    .expiration(expireTime)       // was setExpiration()
    .signWith(signingKey)
    .compact();

// Token parsing
Jwts.parser().verifyWith(signingKey).build()  // was parserBuilder().setSigningKey()
    .parseSignedClaims(token)                 // was parseClaimsJws()
    .getPayload().getSubject();               // was getBody()
```

### Key size requirements

| Algorithm | Minimum key size |
|-----------|-----------------|
| HS256     | 32 bytes         |
| HS384     | 48 bytes         |
| HS512     | 64 bytes         |

JJWT 0.12.x enforces these minimums. Fail fast with a clear error rather than padding.

---

## 7. DGS / GraphQL Migration

### Version upgrade (if using Netflix DGS)

```groovy
// Before
implementation 'com.netflix.graphql.dgs:graphql-dgs-spring-boot-starter:4.9.21'

// After — use the platform BOM + spring-graphql starter
implementation platform('com.netflix.graphql.dgs:graphql-dgs-platform-dependencies:8.7.1')
implementation 'com.netflix.graphql.dgs:graphql-dgs-spring-graphql-starter'
```

### DGS codegen plugin

```groovy
// Before
id "com.netflix.dgs.codegen" version "5.0.6"

// After
id "com.netflix.dgs.codegen" version "6.2.1"
```

### Exception handler changes (graphql-java 21.x)

```java
// BEFORE
@Override
public DataFetcherExceptionHandlerResult onException(
    DataFetcherExceptionHandlerParameters params) {
    // ...
    return DataFetcherExceptionHandlerResult.newResult().error(error).build();
}

// AFTER — method renamed, returns CompletableFuture
@Override
public CompletableFuture<DataFetcherExceptionHandlerResult> handleException(
    DataFetcherExceptionHandlerParameters params) {
    // ...
    return CompletableFuture.completedFuture(
        DataFetcherExceptionHandlerResult.newResult().error(error).build());
}
```

### PageInfo type conflict

If your schema defines custom connection types (e.g., `ArticlesConnection`), the generated `io.spring.graphql.types.PageInfo` may conflict with `graphql.relay.DefaultPageInfo`. Use the generated type directly:

```java
// Before
graphql.relay.PageInfo pageInfo = new DefaultPageInfo(startCursor, endCursor, hasPrev, hasNext);

// After — use the DGS-generated type
io.spring.graphql.types.PageInfo pageInfo = PageInfo.newBuilder()
    .startCursor(startCursor)
    .endCursor(endCursor)
    .hasPreviousPage(hasPrev)
    .hasNextPage(hasNext)
    .build();
```

### Spring GraphQL schema inspection

Add to `application.properties` if using custom relay-style connection types:

```properties
spring.graphql.schema.inspection.enabled=false
```

---

## 8. Spring MVC / Web Changes

### ResponseEntityExceptionHandler signature change

```java
// BEFORE (Spring 5)
@Override
protected ResponseEntity<Object> handleMethodArgumentNotValid(
    MethodArgumentNotValidException ex, HttpHeaders headers,
    HttpStatus status, WebRequest request) { ... }

// AFTER (Spring 6) — HttpStatus → HttpStatusCode
@Override
protected ResponseEntity<Object> handleMethodArgumentNotValid(
    MethodArgumentNotValidException ex, HttpHeaders headers,
    HttpStatusCode status, WebRequest request) { ... }
```

---

## 9. Test Dependencies & Fixes

### rest-assured

```groovy
// 4.x → 5.x (Jakarta servlet support)
testImplementation 'io.rest-assured:rest-assured:5.4.0'
testImplementation 'io.rest-assured:spring-mock-mvc:5.4.0'
```

### Mockito

```groovy
// 4.x → 5.x
testImplementation 'org.mockito:mockito-inline:5.2.0'
```

### Selenium WebDriverWait

```java
// Before (Selenium 3.x / early 4.x)
new WebDriverWait(driver, 10);  // long seconds

// After (Selenium 4.x)
new WebDriverWait(driver, Duration.ofSeconds(10));  // java.time.Duration
```

---

## 10. Other Common Dependency Upgrades

| Dependency                        | 2.x Version    | 3.x Version    |
|-----------------------------------|----------------|----------------|
| MyBatis Spring Boot Starter       | 2.2.x          | 3.0.3          |
| SQLite JDBC                       | 3.36.x         | 3.45.x         |
| Joda-Time                         | 2.10.x         | 2.12.x         |
| JaCoCo                            | 0.8.7          | 0.8.11         |
| Spotless                          | 6.2.x          | 6.25.x         |

---

## 11. Verification Checklist

Run each step in order. Do not proceed until the current step passes.

```bash
# 1. Compile
./gradlew compileJava

# 2. Format check
./gradlew spotlessCheck
# If failures: ./gradlew spotlessApply

# 3. Run tests (skip JaCoCo coverage gate if needed)
./gradlew test -x jacocoTestCoverageVerification

# 4. Full build
./gradlew build -x jacocoTestCoverageVerification

# 5. Verify app starts
./gradlew bootRun
# Ctrl+C after confirming startup

# 6. Verify no stale javax imports
grep -rn "import javax\." src/ --include="*.java" \
  | grep -v "javax\.crypto\." \
  | grep -v "javax\.net\." \
  | grep -v "javax\.sql\." \
  | grep -v "javax\.xml\."
# Should return empty
```

---

## 12. Troubleshooting

### "No node type for 'XxxConnection'"

Spring GraphQL schema inspection fails on custom relay-style types. Fix:

```properties
spring.graphql.schema.inspection.enabled=false
```

### "UnsupportedKeyException: Unable to determine a suitable MAC or Signature algorithm"

JJWT 0.12.x enforces minimum key sizes. Ensure your `jwt.secret` is >= 64 bytes for HS512 (32 for HS256, 48 for HS384).

### Spotless + DGS codegen task dependency error in Gradle 8.x

Gradle 8.x enforces strict task dependency validation. If you see "Task ':spotlessJava' uses output of ':generateJava' without declaring dependency", run `spotlessApply` and `test` as separate Gradle invocations.

### "WebSecurityConfigurerAdapter cannot be resolved"

Removed in Spring Security 6. Replace with `@Bean SecurityFilterChain` (see Section 5).

### "method does not override or implement a method from a supertype" in ExceptionHandler

`ResponseEntityExceptionHandler.handleMethodArgumentNotValid` parameter changed from `HttpStatus` to `HttpStatusCode` (see Section 8).

### "incompatible types: long cannot be converted to Duration"

Selenium 4.x `WebDriverWait` constructor requires `java.time.Duration` (see Section 9).

---

## Version Compatibility Matrix

| Spring Boot | Spring Framework | Spring Security | Java   | Gradle |
|-------------|-----------------|-----------------|--------|--------|
| 2.6.x       | 5.3.x           | 5.6.x           | 11+    | 7.x    |
| 2.7.x       | 5.3.x           | 5.7.x           | 11+    | 7.x    |
| 3.0.x       | 6.0.x           | 6.0.x           | 17+    | 7.5+   |
| 3.1.x       | 6.0.x           | 6.1.x           | 17+    | 7.5+   |
| 3.2.x       | 6.1.x           | 6.2.x           | 17+    | 8.x    |

| DGS Framework | Spring Boot | graphql-java | Starter artifact                          |
|----------------|-------------|--------------|-------------------------------------------|
| 4.x            | 2.x         | 16.x–17.x   | `graphql-dgs-spring-boot-starter`         |
| 7.x            | 3.0–3.1     | 20.x         | `graphql-dgs-spring-graphql-starter`      |
| 8.x            | 3.2         | 21.x         | `graphql-dgs-spring-graphql-starter`      |
