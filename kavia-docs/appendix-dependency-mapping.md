# Appendix: Dependency, Plugin, and API Mapping for Java 21 + Spring Boot 3

## Overview

This appendix lists precise dependency and plugin mappings required for migrating the backend from Spring Boot 2.7.x / Java 11 to Spring Boot 3.3.x / Java 21. It also summarizes Jakarta namespace changes and Spring Security migration notes.

## Dependency Mapping (Old → New)

| Concern | Old (Boot 2.7.x) | New (Boot 3.3.x / Java 21) | Notes |
|--------|-------------------|-----------------------------|-------|
| Spring Boot Parent | org.springframework.boot:spring-boot-starter-parent:2.7.5 | org.springframework.boot:spring-boot-starter-parent:3.3.x | Aligns Spring ecosystem to Spring 6/Jakarta |
| Java Version | 11 | 21 | Use maven-compiler-plugin with release 21 |
| Web | spring-boot-starter-web | spring-boot-starter-web | No change |
| Data JPA | spring-boot-starter-data-jpa | spring-boot-starter-data-jpa | No change (Jakarta annotations in entities) |
| Security | spring-boot-starter-security | spring-boot-starter-security | Migrate to SecurityFilterChain bean |
| Validation | (implicit) javax.validation.* | spring-boot-starter-validation (jakarta.validation.*) | Ensure imports updated |
| MySQL Driver | com.mysql:mysql-connector-j:8.0.x | com.mysql:mysql-connector-j (managed by Boot) | Keep runtime scope |
| MongoDB | spring-boot-starter-data-mongodb | spring-boot-starter-data-mongodb | Optional |
| H2 | com.h2database:h2 (test scope) | com.h2database:h2 (runtime for dev) | Dev-only profile |
| OpenAPI/Swagger | org.springdoc:springdoc-openapi-ui:1.7.0 | org.springdoc:springdoc-openapi-starter-webmvc-ui:2.5.0 | New artifact and autoconfig |
| Lombok | org.projectlombok:lombok:1.18.30 | org.projectlombok:lombok:1.18.30+ | Optional scope |
| JWT | io.jsonwebtoken:jjwt:0.9.1 | io.jsonwebtoken:jjwt-api:0.11.5 + jjwt-impl:0.11.5 + jjwt-jackson:0.11.5 | API changes (parserBuilder) |
| Faker | com.github.javafaker:javafaker:1.0.2 | com.github.javafaker:javafaker:1.0.2 | Dev-only |

## Plugin Mapping

| Plugin | Old | New | Notes |
|--------|-----|-----|------|
| spring-boot-maven-plugin | Managed by Boot 2.7.x | Managed by Boot 3.3.x | No special config required |
| maven-compiler-plugin | 3.10.1 (source/target 11) | 3.11.0 (release 21) | Prefer <release> over source/target |
| maven-surefire-plugin | Boot managed | Boot managed | JUnit 5 |
| maven-failsafe-plugin | Optional | Optional | If integration tests added |

Example compiler config:
```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.11.0</version>
  <configuration>
    <release>21</release>
  </configuration>
</plugin>
```

## Jakarta Namespace Migration

- javax.persistence.* → jakarta.persistence.*
- javax.validation.* → jakarta.validation.*
- javax.servlet.* is typically not used directly; if present, migrate to jakarta.servlet.*

Checklist:
- Update entity imports: Employee.java, Department.java, User.java
- Update any validation annotations if present (e.g., @NotNull → jakarta.validation.constraints.NotNull)
- Rebuild after updates to confirm no javax.* remains

## Spring Security 6 Migration Notes

- Remove WebSecurityConfigurerAdapter
- Provide @Bean SecurityFilterChain to configure HTTP security
- Configure stateless session for JWT:
  - .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
- Expose @Bean AuthenticationManager via AuthenticationConfiguration
- Register JwtRequestFilter with http.addFilterBefore(jwtRequestFilter, UsernamePasswordAuthenticationFilter.class)
- Permit unauthenticated access to: /authenticate, /register, /verify-username/**, /reset-password, /v3/api-docs/**, /swagger-ui/**, /swagger-ui.html

## JJWT 0.11.5 Migration Notes

- Split dependencies: jjwt-api, jjwt-impl (runtime), jjwt-jackson (runtime)
- Instantiate parser via:
  - Jwts.parserBuilder().setSigningKey(key).build()
- Signing key:
  - Use Keys.hmacShaKeyFor(secretBytes) with at least 256-bit secret for HS256
- Validate:
  - Catch io.jsonwebtoken.security.SignatureException for invalid tokens

Example key handling:
```java
import io.jsonwebtoken.security.Keys;
import java.nio.charset.StandardCharsets;
import java.security.Key;

private Key key() {
  return Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
}
```

## OpenAPI with Springdoc 2.x

- Dependency: springdoc-openapi-starter-webmvc-ui
- Access:
  - Swagger UI: /swagger-ui.html
  - OpenAPI JSON: /v3/api-docs
- Existing annotations in controllers remain valid

## H2 Dev Profile Configuration

Recommended dev profile (application-dev.yml):
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

## Validation and Error Parity

- Keep controller exception handling and ResourceNotFoundException behavior unchanged
- Maintain status codes: 200/201/204/400/401/404 as defined in current implementation
- Verify JSON fields and types via curl scripts and regression tests

---
Use this appendix alongside the main migration plan to upgrade dependencies and APIs confidently while preserving all functional behavior.
