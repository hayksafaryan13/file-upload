# File Upload Service (university training project)

The full specification is in `docs/requirements.md`, a word-for-word copy of `docs/File_Upload_Service_Requirements.pdf` (the PDF wins if they ever disagree). Read it before any task. Refer to requirements by ID (FR-xx, NFR-xx, U-xx, S-xx, D-xx). If a requirement is ambiguous, ask me instead of guessing.

Never edit `docs/requirements.md` or the PDF. They are the instructor's specification.

The requirements write the properties path as `file-upload/src/main/resources/application.properties`. Here `file-upload/` is this project folder itself, so the file is `src/main/resources/application.properties`. Do not create a nested `file-upload` folder.

I am a student learning Spring Boot. Explain non-obvious decisions briefly so I can defend the code in a demo.

## Fixed constraints (do not change without asking)

- Java 21, Gradle (Groovy DSL), Spring Boot 4.1.2, base package `com.example.fileupload`
- Packaged as `file-upload.war`, deployed to external Tomcat 11.0.x, sample context `/file-upload`
- Dependencies: `spring-boot-starter-webmvc`, `spring-boot-starter-tomcat-runtime` (providedRuntime), `springdoc-openapi-starter-webmvc-ui:3.2.0`, plus the generated test dependencies and the `jacoco` plugin. Ask before adding any other library.
- The filesystem is the only store (no database). IDs: `f_<epochMillis>_<shortUuid>`. Bytes plus a JSON sidecar, staged then committed.
- Error JSON uses only these codes: `INVALID_REQUEST`, `EMPTY_FILE`, `MAX_SIZE_EXCEEDED`, `UNSUPPORTED_TYPE`, `NOT_FOUND`, `INTERNAL_ERROR`.
- `src/main/resources/application.properties` must match NFR-03 exactly, with the `spring.config.import` line first.

## Environment

- Windows, IntelliJ IDEA, PowerShell terminal. Use `.\gradlew` (not `./gradlew`).
- Fast check: `.\gradlew test`
- Full delivery check: `.\gradlew clean check bootWar` (must produce `build/libs/file-upload.war`)

## Working rules

- Spring Boot 4 is new. Trust the existing `build.gradle`, the compiler and the test output over habits from Spring Boot 2 or 3. If an import or class does not resolve, check what is actually on the classpath instead of guessing.
- Work on one step at a time. Propose a short plan first and wait for my approval before editing files.
- Write tests together with the code. Use `@TempDir` or other temporary storage in tests, never `/tmp/uploads`.
- Run `.\gradlew test` before saying a step is finished, and report the actual result.
- Do not commit or push with git. I review and commit myself.
- Do not edit files under `.idea/` or the Gradle wrapper files.
