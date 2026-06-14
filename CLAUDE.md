# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build and Run

Both Maven and Gradle wrappers are provided and both are fully supported.

```bash
# Run the application
./mvnw spring-boot:run
./gradlew bootRun

# Build (includes format validation, checkstyle, and tests)
./mvnw package
./gradlew build

# Run all tests
./mvnw test
./gradlew test

# Run a single test class
./mvnw test -Dtest=OwnerControllerTests
./gradlew test --tests "org.springframework.samples.petclinic.owner.OwnerControllerTests"

# Apply Spring code formatting
./mvnw spring-javaformat:apply
./gradlew formatMain formatTest

# Recompile CSS from SCSS (Maven only, uses "css" profile)
./mvnw package -P css
```

The app starts on <http://localhost:8080>.

## Code Style

The build enforces **spring-javaformat** on every `mvn validate` / `gradle build` run — it will fail if code is not formatted. Always run `spring-javaformat:apply` (or `formatMain`) after making Java changes before committing. Checkstyle additionally runs **nohttp** to reject plain `http://` URLs in source files (use `https://`).

All commits must include a `Signed-off-by` trailer (DCO):
```
Signed-off-by: Your Name <your@email.com>
```

## Database Configuration

Default profile uses **H2 in-memory** (auto-populated via `src/main/resources/db/h2/`). Switch to MySQL or PostgreSQL with:

```bash
# MySQL (needs running MySQL on port 3306)
./mvnw spring-boot:run -Dspring-boot.run.profiles=mysql

# PostgreSQL (needs running PostgreSQL on port 5432)
./mvnw spring-boot:run -Dspring-boot.run.profiles=postgres
```

Start databases via Docker Compose: `docker compose up mysql` or `docker compose up postgres`.

SQL scripts per database are in `src/main/resources/db/{h2,mysql,postgres}/`.

## Architecture

**Package structure** (`org.springframework.samples.petclinic`):
- `model/` — abstract base classes: `BaseEntity` (id), `NamedEntity` (name), `Person` (firstName, lastName). These are `@MappedSuperclass` entities.
- `owner/` — main domain: `Owner`, `Pet`, `Visit`, `PetType` entities; their controllers (`OwnerController`, `PetController`, `VisitController`) and repositories; `PetValidator` and `PetTypeFormatter`.
- `vet/` — `Vet`, `Specialty`, `Vets` (JAXB wrapper) entities; `VetController` serves both HTML and JSON (`/vets.html`, `/vets`). Vet list is **cached** via JCache/Caffeine (`"vets"` cache, configured in `CacheConfiguration`).
- `system/` — `WelcomeController`, `CrashController` (error demo), `CacheConfiguration`, `WebConfiguration`.

**MVC pattern**: Controllers are package-private classes that talk directly to Spring Data JPA repositories — no service layer. Thymeleaf templates live in `src/main/resources/templates/`.

**Validation**: Bean Validation (`@NotBlank`, `@Size`, `@Pattern`) on entity fields; `PetValidator` provides custom Spring `Validator` logic for pet name uniqueness. Forms bind directly to entity objects.

**GraalVM native image**: `PetClinicRuntimeHints` registers resource patterns (`db/*`, `messages/*`) and serialization hints needed for AOT compilation.

## Testing

| Test class | What it tests |
|---|---|
| `*Tests` in `owner/`, `vet/`, `system/` | Slice tests (`@WebMvcTest`, `@DataJpaTest`) |
| `PetClinicIntegrationTests` | Full app with H2, also usable as `main()` in IDE |
| `MySqlIntegrationTests` | Full app with MySQL via Testcontainers (needs Docker) |
| `PostgresIntegrationTests` | Full app with PostgreSQL via Docker Compose |
| `ValidatorTests` | Bean Validation constraint checks on model classes |
| `I18nPropertiesSyncTest` | Ensures all `messages_*.properties` files have the same keys as `messages.properties` |

MySQL and Postgres integration tests are skipped when Docker is unavailable (`@Testcontainers(disabledWithoutDocker = true)`).
