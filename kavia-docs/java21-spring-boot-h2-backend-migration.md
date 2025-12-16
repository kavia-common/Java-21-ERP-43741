# Migration Plan: Employee Management Backend to Java 21 + Spring Boot 3 (H2)

## Executive Summary and Goals

This document provides a comprehensive, end-to-end migration plan to port the Spring Boot backend from Employee-Management-Fullstack-App/backend to a Java 21-compatible Spring Boot 3.x backend inside the Java-21-ERP-43741 repository. The target runtime is Java 21 with Spring Boot 3.x and H2 for the primary development profile, while preserving 100% functionality, API contracts, architecture, and low-level design (LLD). The plan introduces aligned profiles for MySQL and MongoDB to maintain parity with the current behavior and environment options.

Primary goals:
- Preserve the exact package structure and layered architecture (controller → service → repository → model).
- Maintain all existing REST endpoints and JSON contracts.
- Keep the same authentication and authorization behavior (JWT), but upgrade to Spring Security 6 compatible configuration.
- Provide dev profile using H2 (with MySQL mode for SQL compatibility), and production-like profiles using MySQL and MongoDB.
- Upgrade build, dependencies, and plugins to Java 21 / Spring Boot 3.x compatible versions.
- Deliver complete scaffolding, runbooks, acceptance criteria, and verification steps.

## Constraints and Non-Goals

Constraints:
- No functional changes: same endpoints, same JSON schemas, same error codes, same pagination/sorting if any.
- Same DTOs and models (Department, Employee, User).
- Same authentication behavior: JWT bearer flows (/register, /authenticate, /verify-username, /reset-password), with the same responses and semantics.
- Same business logic, including data initialization for dev.

Non-goals:
- No UI/frontend changes.
- No database model redesign.
- No refactoring of domain logic beyond the minimal required migration fixes (Jakarta imports, Spring Security configuration, JWT library update).

## Target Runtime, Framework, and Dependencies

- Java: 21 (Temurin or OpenJDK)
- Spring Boot: 3.3.x (recommended)
- Spring Framework: aligned with Spring Boot 3.3.x
- Spring Data JPA, Spring Web, Spring Security 6.x
- Springdoc OpenAPI: springdoc-openapi-starter-webmvc-ui 2.x
- JWT: io.jsonwebtoken (jjwt) 0.11.5 (modularized API: jjwt-api, jjwt-impl, jjwt-jackson)
- H2: Latest compatible with Boot 3.3.x
- Lombok: 1.18.30+ (already aligned)
- Faker: com.github.javafaker:javafaker 1.0.2 (optional dev data)
- Database profiles:
  - dev: H2 in-memory (H2 MySQL mode for compatibility)
  - mysql: MySQL via JDBC driver
  - mongo: MongoDB Spring Data (if in use; optional profile)

Dependency alignment recommendations:
- Replace springdoc-openapi-ui 1.x with springdoc-openapi-starter-webmvc-ui 2.x.
- Replace jjwt 0.9.1 with 0.11.5 and adopt the new parserBuilder() API and signing key handling.

H2 Configuration:
- Use H2 in-memory for dev: jdbc:h2:mem:emdb;MODE=MySQL;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
- Enable H2 console in dev profile for troubleshooting: spring.h2.console.enabled=true

## Repository and Module Mapping

Source repository: Employee-Management-Fullstack-App/backend  
Target repository: Java-21-ERP-43741

Preserve the package: com.example.employeemanagement

Proposed target structure inside Java-21-ERP-43741:
- Java-21-ERP-43741/
  - backends/
    - ems-backend/ (new Spring Boot module)
      - pom.xml
      - src/main/java/com/example/employeemanagement/...
        - controller/ (AuthController, EmployeeController, DepartmentController, HomeController)
        - service/ (EmployeeService, DepartmentService)
        - repository/ (EmployeeRepository, DepartmentRepository, UserRepository)
        - model/ (Employee, Department, User)
        - security/ (JwtTokenUtil, JwtRequestFilter, CustomUserDetailsService, Security config)
        - config/ (CorsConfig, DataInitializer)
        - exception/ (ResourceNotFoundException)
      - src/main/resources/
        - application.yml
        - application-dev.yml (H2)
        - application-mysql.yml
        - application-mongo.yml
      - src/test/java/com/example/employeemanagement/... (JUnit tests migrated)

Mirroring rule:
- Keep all Java packages, class names, and method signatures unchanged, except for minimal fixes required by Spring Boot 3 (Jakarta imports, Security configuration bean style) and jjwt 0.11.5.

## Detailed Step-by-Step Migration Procedure

### A) Baseline Inventory (from current backend)

Core modules and packages (from Employee-Management-Fullstack-App/backend):
- pom.xml: Spring Boot 2.7.5, Java 11, jjwt 0.9.1, springdoc 1.7.0, MySQL, MongoDB, H2 (test scope)
- application.properties: MySQL and Mongo URI via variables; JPA settings; server.port=8080
- Packages (com.example.employeemanagement):
  - controller: AuthController, EmployeeController, DepartmentController, HomeController
  - service: EmployeeService, DepartmentService
  - repository: EmployeeRepository, DepartmentRepository, UserRepository
  - model: Employee, Department, User
  - security: CustomUserDetailsService, JwtRequestFilter, JwtTokenUtil, SecurityConfig (WebSecurityConfigurerAdapter)
  - config: CorsConfig, DataInitializer (CommandLineRunner)
  - exception: ResourceNotFoundException (present in source tree per structure)
- OpenAPI: openapi.yaml enumerates endpoints and schemas
- Tests: JUnit 5-based tests under src/test/java/com/example/employeemanagement (CRUD repository tests, API-layer tests)

### B) Create New Backend Module Scaffold in Java-21-ERP-43741

- Create backends/ems-backend as a Maven module with Spring Boot 3.3.x parent.
- Configure Java 21 in the Maven compiler and maven-jar plugin.
- Layout src/main/java with preserved package com.example.employeemanagement.
- Add src/main/resources with application.yml and profile-specific application-*.yml files.

Directory tree (to be created):
- backends/ems-backend/
  - pom.xml
  - src/main/java/com/example/employeemanagement/...
  - src/main/resources/
    - application.yml
    - application-dev.yml
    - application-mysql.yml
    - application-mongo.yml
  - src/test/java/com/example/employeemanagement/...

### C) Copy/Port Code in Phases

Order of copy to reduce compile breakages:

1) Domain models:
- model/Employee.java, model/Department.java, model/User.java
- REQUIRED UPDATE: javax.persistence.* → jakarta.persistence.* imports

2) Repositories:
- repository/EmployeeRepository.java, DepartmentRepository.java, UserRepository.java
- No functional change; ensure imports compile with Boot 3.

3) Services:
- EmployeeService.java, DepartmentService.java
- Maintain method signatures and transactional semantics.

4) Controllers:
- EmployeeController.java, DepartmentController.java, AuthController.java, HomeController.java
- Preserve request mappings, payloads, and responses.

5) Security:
- CustomUserDetailsService.java, JwtRequestFilter.java, JwtTokenUtil.java
- Replace SecurityConfig (WebSecurityConfigurerAdapter) with a SecurityFilterChain bean.

6) Config:
- CorsConfig.java, DataInitializer.java
- DataInitializer should be active only for dev/H2 profile; gate it with @Profile or conditional property.

7) Exception:
- ResourceNotFoundException.java (ensure package and behavior unchanged).

### D) Update Build Files (pom.xml) for Java 21

Parent and properties:
- Use Spring Boot 3.3.x parent
- Set java.version=21
- Add maven-compiler-plugin with release 21 (or toolchains)

Dependencies:
- spring-boot-starter-web
- spring-boot-starter-data-jpa
- spring-boot-starter-security
- spring-boot-starter-validation (Jakarta validation)
- com.h2database:h2 (no test scope; enabled by dev profile)
- com.mysql:mysql-connector-j (runtime for mysql profile)
- org.springframework.boot:spring-boot-starter-data-mongodb (for mongo profile)
- org.springdoc:springdoc-openapi-starter-webmvc-ui:2.5.0
- io.jsonwebtoken:jjwt-api:0.11.5
- io.jsonwebtoken:jjwt-impl:0.11.5 (runtime)
- io.jsonwebtoken:jjwt-jackson:0.11.5 (runtime)
- com.github.javafaker:javafaker:1.0.2 (dev only or optional)
- org.projectlombok:lombok (provided)
- test: spring-boot-starter-test

Plugins:
- spring-boot-maven-plugin
- maven-compiler-plugin release 21

### E) Profiles: dev (H2), mysql, mongo

- dev: H2 in-memory with MySQL mode; enable data initializer; typical port 8080
- mysql: External MySQL connection via env vars; disable H2; keep same JPA props
- mongo: Enable Spring Data Mongo; if not actively used by repositories, keep optional

### F) Configuration Migration (application.yml/properties)

- Convert application.properties into application.yml with profile splits
- Migrate to Jakarta naming where needed
- Add spring.h2.console.enabled=true in dev

### G) Security/JWT Alignment

- Replace WebSecurityConfigurerAdapter with @Bean SecurityFilterChain
- Configure stateless session, JWT filter oncePerRequest
- Allow: /authenticate, /register, /verify-username/**, /reset-password; secure /api/**
- jjwt upgrade: use parserBuilder() and Keys.hmacShaKeyFor for secret handling

### H) OpenAPI/Swagger Configuration

- springdoc-openapi-starter-webmvc-ui 2.x automatically serves Swagger UI at /swagger-ui.html and /v3/api-docs
- Preserve endpoint metadata via annotations already present

### I) Test Migration (JUnit)

- Spring Boot 3 uses JUnit 5 by default
- Ensure tests import org.junit.jupiter.* and use @SpringBootTest or @DataJpaTest as needed
- If H2 is used for tests, set profile to dev during test or include test config

### J) CI Build Updates

- Use Maven with Java 21 toolchain
- mvn -B -ntp -DskipTests=false clean verify
- For multi-profile builds, test dev (H2) by default

### K) Local Run Commands

- Dev (H2): mvn spring-boot:run -Dspring-boot.run.profiles=dev
- MySQL: mvn spring-boot:run -Dspring-boot.run.profiles=mysql
- Mongo: mvn spring-boot:run -Dspring-boot.run.profiles=mongo

## Database Strategy

Primary dev profile: H2
- URL: jdbc:h2:mem:emdb;MODE=MySQL;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
- User: sa, empty password
- Schema generation: spring.jpa.hibernate.ddl-auto=update
- Show SQL: true for dev

MySQL profile: Parity with current app
- Use env vars for URL, user, password, ssl-mode
- spring.jpa.hibernate.ddl-auto=update
- Ensure dialect auto-detection or set to org.hibernate.dialect.MySQLDialect

Mongo profile: Optional parity
- spring.data.mongodb.uri from env
- Only enable if you have Mongo repositories or want to keep operational parity

Data migration notes:
- Dev seeding via DataInitializer should be profile-guarded to avoid wiping production data
- For MySQL profile, use migrations or keep update with caution

## Compatibility Preservation Plan

Contracts preserved:
- Paths and methods: /api/employees, /api/departments, /authenticate, /register, /verify-username/{username}, /reset-password
- JSON payloads: Employee, Department, User fields unchanged
- Error codes: Maintain 200/201/204/400/401/404 semantics as per current controllers
- Pagination/sorting: If none existed, do not introduce; if any existed, keep behavior
- Validation messages: Ensure Jakarta validation annotations map to same rules

CORS:
- Preserve permissive CORS policy (allowedOriginPatterns "*") as in CorsConfig for dev; tighten in prod as needed.

## Risk and Rollback Plan

Risks:
- API breakage due to Jakarta imports, security changes, jjwt API changes
- Springdoc changes might shift Swagger URL defaults
- DataInitializer accidentally wiping data if not profile-guarded

Mitigation:
- Create a migration branch in Java-21-ERP-43741; do not decommission original repo until acceptance tests pass
- Enable dev-only DataInitializer; disable for mysql/mongo profiles
- Write regression tests for endpoints and payloads; compare against openapi.yaml

Rollback:
- Keep the original backend intact
- If regression found, revert the module or switch profile to original stack while fixing issues
- Maintain git tags for each migration milestone

## Acceptance Criteria

- Builds and runs on Java 21 with Spring Boot 3.3.x (dev profile with H2)
- All endpoints function with identical behavior and response contracts as defined in openapi.yaml
- JWT authentication works: /register, /authenticate issue valid tokens; protected endpoints require Bearer token
- Swagger UI available at /swagger-ui.html and OpenAPI at /v3/api-docs
- Repository tests pass using H2 dev profile
- Manual curl script succeeds end-to-end
- CI pipeline green for dev profile

## Completion Checklist and Progress Table

Checklist:
- Create new module scaffold in Java-21-ERP-43741
- Port domain, repository, service, controller, config, exception code
- Migrate security to Spring Security 6 (SecurityFilterChain)
- Upgrade jjwt to 0.11.5 and adapt JwtTokenUtil
- Replace javax.* with jakarta.* imports in JPA and validation
- Migrate application.properties → application.yml with profiles
- Add springdoc starter 2.x and verify Swagger UI
- Add dev H2 config and seed data guarded by dev profile
- Port/migrate tests to run over H2
- Add runbook and CI commands

Progress table (example):
| Item | Owner | Status |
|------|-------|--------|
| Module scaffold (ems-backend) | Backend Eng | Not Started |
| Domain + Repos port | Backend Eng | Not Started |
| Services + Controllers port | Backend Eng | Not Started |
| Security migration | Backend Eng | Not Started |
| Config/profiles migration | Backend Eng | Not Started |
| Tests migrated | QA | Not Started |
| CI updated | DevOps | Not Started |
| Verification & Acceptance | QA | Not Started |

## Basic Scaffolding Instructions

### Directory Tree

- Java-21-ERP-43741/
  - backends/
    - ems-backend/
      - pom.xml
      - src/main/java/com/example/employeemanagement/...
      - src/main/resources/
        - application.yml
        - application-dev.yml
        - application-mysql.yml
        - application-mongo.yml
      - src/test/java/com/example/employeemanagement/...

### Initial pom.xml (fragment for Boot 3 + Java 21)

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.2</version>
    <relativePath/>
  </parent>

  <groupId>com.example</groupId>
  <artifactId>ems-backend</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <name>ems-backend</name>

  <properties>
    <java.version>21</java.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- H2 for dev -->
    <dependency>
      <groupId>com.h2database</groupId>
      <artifactId>h2</artifactId>
      <scope>runtime</scope>
    </dependency>

    <!-- MySQL driver -->
    <dependency>
      <groupId>com.mysql</groupId>
      <artifactId>mysql-connector-j</artifactId>
      <scope>runtime</scope>
    </dependency>

    <!-- MongoDB (optional profile) -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>

    <!-- OpenAPI -->
    <dependency>
      <groupId>org.springdoc</groupId>
      <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
      <version>2.5.0</version>
    </dependency>

    <!-- JWT (0.11.5 modular) -->
    <dependency>
      <groupId>io.jsonwebtoken</groupId>
      <artifactId>jjwt-api</artifactId>
      <version>0.11.5</version>
    </dependency>
    <dependency>
      <groupId>io.jsonwebtoken</groupId>
      <artifactId>jjwt-impl</artifactId>
      <version>0.11.5</version>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>io.jsonwebtoken</groupId>
      <artifactId>jjwt-jackson</artifactId>
      <version>0.11.5</version>
      <scope>runtime</scope>
    </dependency>

    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
    </dependency>

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.11.0</version>
        <configuration>
          <release>21</release>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

### application.yml Examples

application.yml:
```yaml
spring:
  application:
    name: employee-management
server:
  port: 8080

# Default profile can be dev
spring:
  profiles:
    default: dev
```

application-dev.yml (H2):
```yaml
spring:
  datasource:
    url: jdbc:h2:mem:emdb;MODE=MySQL;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
    driver-class-name: org.h2.Driver
    username: sa
    password:
  h2:
    console:
      enabled: true
      path: /h2-console
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

application-mysql.yml:
```yaml
spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL:jdbc:mysql://localhost:3306/employee_management}
    username: ${SPRING_DATASOURCE_USERNAME:root}
    password: ${SPRING_DATASOURCE_PASSWORD:password}
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

application-mongo.yml:
```yaml
spring:
  data:
    mongodb:
      uri: ${SPRING_DATA_MONGODB_URI:mongodb://localhost:27017/employee_management}
```

### Sample CommandLineRunner (seed - dev only)

```java
@Profile("dev")
@Configuration
public class DevDataInitializer implements CommandLineRunner {
  // same as existing DataInitializer; copy logic and annotate @Profile("dev")
}
```

## Post-Migration Verification

Manual test script (curl):
```bash
# Health: Swagger
curl -i http://localhost:8080/swagger-ui.html

# Register
curl -s -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"password"}'

# Authenticate
TOKEN=$(curl -s -X POST http://localhost:8080/authenticate \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"password"}' | jq -r .token)

# Verify username
curl -i http://localhost:8080/verify-username/alice

# Protected: list employees
curl -s http://localhost:8080/api/employees \
  -H "Authorization: Bearer $TOKEN"

# CRUD sample
curl -s -X POST http://localhost:8080/api/employees \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"firstName":"John","lastName":"Doe","email":"john.doe@example.com","age":30,"department":{"id":1}}'
```

Smoke tests:
- Start app in dev (H2), confirm Swagger loads and all CRUD endpoints work as expected with JWT.
- Validate 401 on protected endpoints without Authorization.
- Verify JSON schemas match openapi.yaml.

Regression guidelines:
- Use existing test classes as reference; port to the new module and run with dev profile.
- Ensure repository CRUD operations perform identically with H2 (MySQL mode).

## Runbook

Build:
- mvn -B -ntp clean verify

Run with profiles:
- Dev (H2): mvn spring-boot:run -Dspring-boot.run.profiles=dev
- MySQL: mvn spring-boot:run -Dspring-boot.run.profiles=mysql
- Mongo: mvn spring-boot:run -Dspring-boot.run.profiles=mongo

Ports:
- Backend: 8080
- Frontend (if used locally): 3000; CORS allowed via CorsConfig

## Known Pitfalls Migrating to Java 21 and Spring Boot 3.x

- Jakarta namespace:
  - javax.persistence.* → jakarta.persistence.*
  - javax.validation.* → jakarta.validation.*
- Spring Security:
  - WebSecurityConfigurerAdapter removed
  - Must define @Bean SecurityFilterChain and AuthenticationManager bean if needed
- jjwt:
  - 0.9.1 → 0.11.5 requires jjwt-api + jjwt-impl + jjwt-jackson
  - Use Jwts.parserBuilder() and Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8))
- Springdoc:
  - Use starter 2.x; artifact coordinates changed
- CORS:
  - Keep CorsConfig with allowedOriginPatterns("*") for dev; refine for prod
- H2:
  - Use MODE=MySQL for compatibility with MySQL-specific SQL if present

## Example Code Changes (Security and JWT)

### Security Configuration (from WebSecurityConfigurerAdapter to SecurityFilterChain)

Before (current SecurityConfig extends WebSecurityConfigurerAdapter):
- Located at security/SecurityConfig.java
- Permits all (temporary)
- Needs migration to Boot 3

After (Boot 3 style):
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

  private final JwtRequestFilter jwtRequestFilter;
  private final UserDetailsService userDetailsService;

  public SecurityConfig(JwtRequestFilter jwtRequestFilter, UserDetailsService userDetailsService) {
    this.jwtRequestFilter = jwtRequestFilter;
    this.userDetailsService = userDetailsService;
  }

  @Bean
  public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
  }

  @Bean
  public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
    return config.getAuthenticationManager();
  }

  @Bean
  public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
      .csrf(csrf -> csrf.disable())
      .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
      .authorizeHttpRequests(auth -> auth
        .requestMatchers("/authenticate", "/register", "/verify-username/**", "/reset-password", "/v3/api-docs/**", "/swagger-ui/**", "/swagger-ui.html").permitAll()
        .anyRequest().authenticated()
      )
      .addFilterBefore(jwtRequestFilter, UsernamePasswordAuthenticationFilter.class);
    return http.build();
  }
}
```

### JwtTokenUtil updated to jjwt 0.11.5

Before (0.9.1):
```java
return Jwts.parser().setSigningKey(secret).parseClaimsJws(token).getBody();
```

After (0.11.5):
```java
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import java.nio.charset.StandardCharsets;
import java.security.Key;

@Component
public class JwtTokenUtil {
  private final String secret = "replace-with-strong-secret-at-least-32-bytes-long........";
  private Key key() {
    // Prefer a base64 secret or use raw bytes with sufficient length
    return Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
  }

  public String extractUsername(String token) {
    return Jwts.parserBuilder().setSigningKey(key()).build().parseClaimsJws(token).getBody().getSubject();
  }

  public Date extractExpiration(String token) {
    return Jwts.parserBuilder().setSigningKey(key()).build().parseClaimsJws(token).getBody().getExpiration();
  }

  public String generateToken(String username) {
    return Jwts.builder()
      .setSubject(username)
      .setIssuedAt(new Date())
      .setExpiration(new Date(System.currentTimeMillis() + 1000L * 60 * 60 * 24 * 7))
      .signWith(key(), SignatureAlgorithm.HS256)
      .compact();
  }

  public boolean validateToken(String token, String username) {
    final String extractedUsername = extractUsername(token);
    return extractedUsername.equals(username) && extractExpiration(token).after(new Date());
  }
}
```

### Jakarta Imports for Entities

Before:
```java
import javax.persistence.*;
```

After:
```java
import jakarta.persistence.*;
```

Validation annotations:
- javax.validation.* → jakarta.validation.*

## Appendix

### Recommended Dependency Mapping (Old → New)

- Spring Boot 2.7.5 → 3.3.x
- javax.* → jakarta.*
- springdoc-openapi-ui:1.7.0 → springdoc-openapi-starter-webmvc-ui:2.5.0
- jjwt:0.9.1 → jjwt-api:0.11.5 + jjwt-impl:0.11.5 + jjwt-jackson:0.11.5
- H2 test-scope → runtime in dev profile

### Example POM Fragments

OpenAPI starter:
```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.5.0</version>
</dependency>
```

JWT:
```xml
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt-api</artifactId>
  <version>0.11.5</version>
</dependency>
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt-impl</artifactId>
  <version>0.11.5</version>
  <scope>runtime</scope>
</dependency>
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt-jackson</artifactId>
  <version>0.11.5</version>
  <scope>runtime</scope>
</dependency>
```

### Example Configuration Files

Already provided in the scaffolding section for application ymls.

### Mermaid: Migration Overview

```mermaid
flowchart LR
  A["Source Backend (Boot 2.7, Java 11)"] --> B["Create new module (Boot 3.3, Java 21)"]
  B --> C["Port domain + repos (jakarta.*)"]
  C --> D["Port services + controllers (no functional change)"]
  D --> E["Migrate Security (SecurityFilterChain + JWT)"]
  E --> F["Profiles: dev(H2), mysql, mongo"]
  F --> G["OpenAPI starter 2.x"]
  G --> H["Tests on H2 (dev)"]
  H --> I["Smoke & Regression via curl + JUnit"]
```

### Mermaid: Runtime Profiles

```mermaid
flowchart TB
  subgraph Dev Profile (H2)
    API1["Spring Boot 3 API"] --> H2["H2 (MySQL mode)"]
  end
  subgraph MySQL Profile
    API2["Spring Boot 3 API"] --> MYSQL["MySQL DB"]
  end
  subgraph Mongo Profile
    API3["Spring Boot 3 API"] --> MONGO["MongoDB"]
  end
```

---
This document outlines a precise, actionable plan to port the backend to Java 21 with Spring Boot 3.x and H2, keeping 100% of the existing behavior and contracts. Follow the procedure step-by-step and use the runbooks and acceptance criteria to validate completion.
