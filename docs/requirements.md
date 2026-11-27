ENGINEERING HANDOFF — REQUIREMENTS / 10 NOV 2026

**01 / FUNCTIONAL REQUIREMENTS**

# File Upload Service

Build a REST API that accepts, validates, stores, lists, and downloads files. This is a backend training project with Swagger UI for manual API testing.

| LANGUAGE & BUILD | FRAMEWORK | DEPLOYMENT |
|---|---|---|
| **Java 21 / Gradle** | **Spring Boot 4.1.2** | **WAR / Tomcat 11.0.x** |

## API endpoints

| ID | METHOD AND PATH | REQUEST AND SUCCESS RESPONSE |
|---|---|---|
| FR-01 | `POST /api/files` | Upload one `file` part. Return **201 Created** with file metadata and a `Location` header matching `downloadUrl`. |
| FR-02 | `POST /api/files/batch` | Upload repeated `files` parts. Return **200 OK** with one result per file in request order. Save each valid file independently, even when other files fail. |
| FR-03 | `GET /api/files` | Return **200 OK** and an array of `id`, `filename`, `size`, and `createdAt`. Sort newest first, then by ID ascending for ties. Return `[]` when empty. |
| FR-04 | `GET /api/files/{id}` | Return **200 OK** with the original bytes, detected `Content-Type`, `Content-Length`, and a safely encoded `Content-Disposition: attachment` filename. |

## Upload validation

**FR-05** Both POST endpoints accept `multipart/form-data`. Single upload requires exactly one file part. Batch requires 1 to 10 file parts by default. Reject missing or unexpected parts, duplicate single-file parts, or an excessive batch count before writing any files.

**FR-06** Check files in this order. Reject zero-byte files, then files larger than the configured per-file maximum, then unsupported types. The default maximum is **2 MiB (2,097,152 bytes)**, inclusive.

**FR-07** Detect file type from bytes and enforce the configured allowlist. Defaults are `image/png`, `image/jpeg`, and `application/pdf`. Accept a missing or generic `application/octet-stream` part type when detection succeeds. Reject a specific declared type that disagrees with the detected type. Reject unknown types.

All paths are relative to the application context. The sample deployment uses `/file-upload`.

## Default size error

```
{"error":"MAX_SIZE_EXCEEDED","message":"File too large. Max 2097152 bytes.",
 "details":{"limitBytes":2097152}}
```

FILE UPLOAD SERVICE — 01 / 09

---

ENGINEERING HANDOFF — REQUIREMENTS / 10 NOV 2026

**02 / FUNCTIONAL REQUIREMENTS**

# Response contract

Upload, list, and error payloads use JSON. Downloads return binary content.

## Successful single upload

```
{
  "id": "f_1760954400000_7fa3b1",
  "filename": "report.pdf",
  "contentType": "application/pdf",
  "size": 524288,
  "downloadUrl": "/file-upload/api/files/f_1760954400000_7fa3b1",
  "createdAt": "2025-10-20T10:00:00Z"
}
```

`size` is a byte count. `createdAt` is an ISO 8601 UTC timestamp. `downloadUrl` includes the deployed context path and can be used on the same origin.

## Batch with one successful file

```
[
  {"filename":"a.png","status":"OK","meta":{
    "id":"f_1760954400000_7fa3b2","filename":"a.png","contentType":"image/png",
    "size":1024,"downloadUrl":"/file-upload/api/files/f_1760954400000_7fa3b2",
    "createdAt":"2025-10-20T10:00:00Z"}},
  {"filename":"b.exe","status":"ERROR","error":"UNSUPPORTED_TYPE",
   "message":"Allowed types: image/png, image/jpeg, application/pdf"}
]
```

Use the single-upload metadata schema for `meta`. A valid, fully parsed batch returns 200 even if every file fails. Request-level failures return one JSON error object.

## Error codes

| CODE | HTTP | WHEN TO USE |
|---|---|---|
| `INVALID_REQUEST` | 400, 415 | 400 for malformed multipart or wrong part count or names. 415 for a request with a non-multipart media type. |
| `EMPTY_FILE` | 400 | A file part contains zero bytes. |
| `MAX_SIZE_EXCEEDED` | 413 | Per-file or multipart request limit. Include `details.limitBytes`. For a request limit also include `details.scope = REQUEST`. |
| `UNSUPPORTED_TYPE` | 415 | File type is disallowed, unknown, or conflicts with a specific declared type. |
| `NOT_FOUND` | 404 | Download ID is invalid or unknown. |
| `INTERNAL_ERROR` | 500 | Unexpected processing or filesystem failure. |

For batch item failures, use the same codes in the item's `error` field. Include `message` and any applicable `details`.

FILE UPLOAD SERVICE — 02 / 09

---

ENGINEERING HANDOFF — REQUIREMENTS / 10 NOV 2026

**03 / PROJECT SETUP**

# Generate the project

Create the Gradle project with Spring Initializr before implementing the API. [10]

[Open Spring Initializr at start.spring.io](https://start.spring.io/)

## 1. Select the project settings

| SETTING | VALUE |
|---|---|
| Project and language | Gradle - Groovy / Java |
| Spring Boot and Java version | 4.1.2 stable / 21 |
| Group and package name | `com.example` / `com.example.fileupload` |
| Artifact and name | `file-upload` for both |
| Packaging and configuration | War / Properties |
| Dependencies | Spring Web |

Click **Generate**, extract the ZIP, and import the folder as a Gradle project using JDK 21. Keep the generated Gradle Wrapper and test configuration.

## 2. Verify the generated project

In `build.gradle`, verify the `war` plugin, the Java 21 toolchain, the Boot plugin version, and the dependencies below. Add the Swagger dependency and set the WAR name. Add `id 'jacoco'` to the existing plugins block for coverage. [1, 2, 7, 8]

```
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webmvc'
    providedRuntime 'org.springframework.boot:spring-boot-starter-tomcat-runtime'
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:3.2.0'
}

tasks.named('bootWar') {
    archiveFileName = 'file-upload.war'
}
```

Verify the generated initializer extends `SpringBootServletInitializer` and registers the main application class. Merge the dependency lines into the existing block and keep the generated test dependencies.

From the project root, run `./gradlew test bootWar`. Confirm that `build/libs/file-upload.war` is present. The final delivery build also runs coverage verification as described in the runtime requirements.

FILE UPLOAD SERVICE — 03 / 09

---

ENGINEERING HANDOFF — REQUIREMENTS / 10 NOV 2026

**04 / RUNTIME REQUIREMENTS**

# Build and configuration

Use Spring MVC and deploy the same Gradle-built WAR to external Tomcat 11.

## Build and deploy

**NFR-01** Pin Spring Boot **4.1.2**, a Java 21 toolchain, and a supported Gradle Wrapper (8.14+ in the 8.x line or 9.x). Use `spring-boot-starter-webmvc`, Gradle's `war` plugin, `SpringBootServletInitializer`, and `spring-boot-starter-tomcat-runtime` in `providedRuntime`. [1, 2]

**NFR-02** `./gradlew clean check bootWar` must produce `build/libs/file-upload.war`. Deploy it to external Tomcat 11.0.x running Java 21 and verify the `/file-upload` context. Spring Boot 4.1 requires Servlet 6.1. [1, 2]

## Configuration defaults

**NFR-03** Put defaults in `file-upload/src/main/resources/application.properties` and keep the import on the first line. The optional `${user.home}/.spring/file-upload.properties` overrides bundled defaults. `user.home` is the home directory of the JVM running Tomcat. [3]

```
spring.config.import=optional:file:${user.home}/.spring/file-upload.properties
spring.application.name=file-upload

files.upload-dir=/tmp/uploads
files.max-file-size=2MB
files.max-request-size=22MB
files.max-batch-files=10
files.allowed-content-types=image/png,image/jpeg,application/pdf

spring.servlet.multipart.file-size-threshold=0B
spring.servlet.multipart.max-file-size=${files.max-request-size}
spring.servlet.multipart.max-request-size=${files.max-request-size}
```

**Precedence for supported inputs, highest first**
JVM `-D` property → environment variable → imported file → bundled defaults. For example, `-Dfiles.max-file-size=3MB` overrides `FILES_MAXFILESIZE=4MB`. Missing external files are allowed. [3]

## Per-file limit and request limit

**NFR-04** Enforce the 2 MiB per-file limit in application validation. Set both multipart parser limits to the 22 MiB request ceiling by default, leaving room for ten maximum-size files and form overhead. A request that hits a parser size limit returns one JSON 413 and saves no files. [4]

Allow for multipart overhead when changing limits. Check Tomcat connector part-count and aborted-upload handling so clients receive the expected JSON response. [5]

FILE UPLOAD SERVICE — 04 / 09

---

ENGINEERING HANDOFF — REQUIREMENTS / 10 NOV 2026

**05 / NON-FUNCTIONAL REQUIREMENTS**

# Storage and quality

A successful upload makes both the file and its metadata available after an application restart.

## Filesystem and security

**NFR-05** Use the filesystem as the sole store. Keep file bytes and metadata outside the deployed web root. Use a JSON sidecar or equivalent metadata record. Generate a stable, unique ID in the format `f_<epochMillis>_<shortUuid>` and use it for storage paths. Treat original filenames as display metadata. Files with the same name must keep separate IDs.

**NFR-06** Stage each upload and publish it only after both file and metadata are complete. Clean up temporary files after failed writes. List and serve only committed records. Concurrent uploads must not overwrite each other.

**NFR-07** Validate IDs before resolving paths and keep all storage access inside the upload directory. Handle path-like filenames safely and prevent control characters from reaching download headers. Stream upload copies and downloads with bounded memory. Return generic server errors and log the cause without file contents or stack traces in responses. [6]

**NFR-08** Validate limits and the allowlist at startup. Require positive size and batch limits, a request ceiling above the per-file limit, and at least one allowed type. Fail with a clear log message if the upload directory cannot be created or written. Use a persistent directory override when files must survive host cleanup of `/tmp`. Run one application instance in a trusted training environment.

**Download failures.** If a committed file record points to missing or unreadable bytes, return `500 INTERNAL_ERROR` when the failure is detected before download headers are sent. Log the file ID and the cause.

## Swagger and OpenAPI

**NFR-09** Pin springdoc-openapi **3.2.0** using `springdoc-openapi-starter-webmvc-ui`. Expose Swagger UI at `/swagger-ui.html` and OpenAPI JSON at `/v3/api-docs`, relative to the deployment context. Document both multipart field names, response schemas, status codes, errors, and the binary download. Verify both upload forms with Try it out. [7]

## Automated tests and coverage

**NFR-10** Use Spring Boot-managed JUnit Jupiter and JaCoCo. Enforce **at least 80% line coverage** across production application classes. Wire verification and HTML report generation into `./gradlew check`. Use temporary storage and include a real HTTP multipart test to exercise parser-level limits. [8, 9]

FILE UPLOAD SERVICE — 05 / 09

---

ENGINEERING HANDOFF — REQUIREMENTS / 10 NOV 2026

**06 / DEVELOPER HANDOFF**

# Manual verification

Deploy the WAR to external Tomcat before running the checks below.

## Start with cURL

Use a valid PDF named report.pdf and a plain-text file named blocked.txt. Set the base URL to the actual deployed context.

```
BASE=http://localhost:8080/file-upload

curl -i -F 'file=@report.pdf;type=application/pdf' "$BASE/api/files"

curl -i -F 'files=@report.pdf;type=application/pdf' \
  -F 'files=@blocked.txt;type=text/plain' "$BASE/api/files/batch"

curl -i "$BASE/api/files"

# Set FILE_ID to the id returned by the first upload.
FILE_ID='<id-from-upload-response>'
curl -fS -D - -o downloaded.pdf "$BASE/api/files/$FILE_ID"
cmp report.pdf downloaded.pdf
```

Expect 201, 200 with mixed results, 200 with a file list, then a successful download. `cmp` should exit successfully.

## Repeat with Postman and Swagger

In Postman, use Body → form-data and select File for `file` or each repeated `files` field. Let Postman generate the multipart boundary. In Swagger UI, execute both upload endpoints and verify the list and download routes in the deployed context.

## Verify the external properties file

Set `files.max-file-size=3MB` in the external file and restart Tomcat. A valid file between 2 and 3 MiB should upload successfully. Remove the override and restart to verify that the same file is rejected under the bundled limit.

## Verify rejection without side effects

Before a request-level failure test, record the list from `GET /api/files`. After the failed request, compare the list and verify that zero records were created. For a mixed batch, verify that the number of new records matches the number of successful items.

FILE UPLOAD SERVICE — 06 / 09

---

ENGINEERING HANDOFF — REQUIREMENTS / 10 NOV 2026

**07 / ACCEPTANCE CRITERIA**

# Upload scenarios

Use real multipart HTTP requests for parser-level tests. Start each case with isolated storage or record the existing file count before sending the request.

| ID | SCENARIO | EXPECTED RESULT |
|---|---|---|
| U-01 | Valid PNG, JPEG, and PDF | Each returns 201 with the detected type, byte count, ID, UTC timestamp, and a `Location` header equal to `downloadUrl`. The returned URL downloads the file. |
| U-02 | Empty file and size boundary | An empty part returns `400 EMPTY_FILE`. Valid fixtures of 2,097,151 and 2,097,152 bytes pass. A file of 2,097,153 bytes returns `413 MAX_SIZE_EXCEEDED` with the per-file limit in bytes. |
| U-03 | Spoofed or generic file type | Plain text named `report.pdf` or a JPEG declared as PNG returns `415 UNSUPPORTED_TYPE`. A valid PDF with an omitted or generic part type is accepted. |
| U-04 | Malformed or wrong request shape | Missing or unexpected parts, duplicate single-upload parts, and a malformed boundary return `400 INVALID_REQUEST`. A non-multipart request returns `415 INVALID_REQUEST`. Both cases create zero records. |
| U-05 | Batch count and upper bound | One file and ten files are accepted, including ten files at the per-file maximum in one normal request. Eleven small files return `400 INVALID_REQUEST` and create zero records. |
| U-06 | Mixed and all-invalid batches | Both return 200 and one result per part in order. Include an oversized file, an empty file and an unsupported type. Only valid files appear in the list. |
| U-07 | Request ceiling with valid parts | Set `files.max-request-size=3MB` and send two valid 2 MiB files in one batch. Expect one JSON 413 `MAX_SIZE_EXCEEDED` with `scope=REQUEST` and `limitBytes=3145728` in details. Zero records are created. |
| U-08 | Interrupted upload and retry | Abort the connection before the multipart body is complete. The partial upload stays out of the file list. Retrying with a complete request succeeds. |

FILE UPLOAD SERVICE — 07 / 09

---

ENGINEERING HANDOFF — REQUIREMENTS / 10 NOV 2026

**08 / ACCEPTANCE CRITERIA**

# Storage and deployment

Exercise storage failures with a test double or a temporary directory. Run deployment checks on external Tomcat 11 with the WAR produced by Gradle.

| ID | SCENARIO | EXPECTED RESULT |
|---|---|---|
| S-01 | Empty list and sort order | Fresh storage returns 200 with `[]`. Seed distinct and identical timestamps. The list sorts newest first and by ID ascending for timestamp ties. |
| S-02 | Download and attachment filename | Download matches the uploaded bytes, type and length. A filename with spaces and Unicode is encoded safely in the attachment header. Path-like names and control characters cannot affect the storage path or inject response headers. |
| S-03 | Unknown ID and traversal input | A syntactically valid unknown ID and a malformed ID that reaches the app return `404 NOT_FOUND`. A traversal-shaped URL is denied by Tomcat or the app. All filesystem lookups remain inside the upload root. |
| S-04 | Concurrent uploads with the same name | Send two simultaneous uploads with the same filename and different contents. Both succeed with distinct IDs. Downloading each ID returns its own bytes. |
| S-05 | Storage write fails mid-upload | Simulate a write failure after staging starts. Single upload returns `500 INTERNAL_ERROR`. In a batch, HTTP 200 includes an `ERROR` item with `INTERNAL_ERROR`. Other valid items are saved. Staging files are cleaned up and the failed item is absent from the list. |
| S-06 | Restart and incomplete storage | After restart, existing IDs and bytes are available and leftover staging entries stay hidden. A committed record with missing bytes returns `500 INTERNAL_ERROR` before streaming. |
| S-07 | Configuration and startup | Verify packaged defaults with the external file absent, then file, environment and JVM overrides in precedence order. Invalid limits, an empty allowlist or an unwritable directory fail startup with a clear log. |
| S-08 | WAR context and Swagger | Verify both upload forms and the OpenAPI JSON on Tomcat. Redeploy under a second context, such as `/file-upload-test`, and check that both the download URL and Location reflect that context. |
| S-09 | Coverage gate | The Gradle check task generates the JaCoCo report and fails when measured application line coverage falls below 80%. |

FILE UPLOAD SERVICE — 08 / 09

---

ENGINEERING HANDOFF — REQUIREMENTS / 10 NOV 2026

**09 / DEVELOPER HANDOFF**

# Delivery checklist

Submit a build that another developer can run and deploy from the project README.

**D-01** Submit the Gradle project with wrapper files, source code, automated tests, and a README covering build, external Tomcat deployment, configuration overrides, cURL and Postman, and Swagger URLs.

**D-02** Provide the WAR and JaCoCo HTML report. Demonstrate the deployed API, including a mixed batch and a download after restarting the application.

**D-03** Record which acceptance scenarios passed in automated tests and which were verified on external Tomcat.

## Implementation references

- [[1] Spring Boot system requirements](https://docs.spring.io/spring-boot/system-requirements.html)
- [[2] Spring Boot traditional WAR deployment](https://docs.spring.io/spring-boot/how-to/deployment/traditional-deployment.html)
- [[3] Spring Boot externalized configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html)
- [[4] Spring Boot multipart properties](https://docs.spring.io/spring-boot/appendix/application-properties/index.html)
- [[5] Apache Tomcat 11 HTTP connector settings](https://tomcat.apache.org/tomcat-11.0-doc/config/http.html)
- [[6] OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [[7] springdoc 3.2.0 release](https://github.com/springdoc/springdoc-openapi/releases/tag/v3.2.0) / [Swagger setup and URLs](https://springdoc.org/getting-started.html)
- [[8] Gradle JaCoCo plugin and coverage verification](https://docs.gradle.org/current/userguide/jacoco_plugin.html)
- [[9] Spring Framework MockMvc multipart request behavior](https://docs.spring.io/spring-framework/reference/testing/mockmvc/hamcrest/requests.html)
- [[10] Spring Initializr](https://start.spring.io/) / [Spring Boot project setup guide](https://spring.io/guides/gs/spring-boot/)

FILE UPLOAD SERVICE — 09 / 09
