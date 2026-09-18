# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

There is **no application code here**. This repo is a *generator configuration* project: OpenAPI specs + a customised OpenAPI Generator template set + GitHub Actions workflows that produce Java (Jersey3) API clients *and Mockito/Spring mock providers*, then publish them as Maven artifacts to GitHub Packages.

Everything under `generated-client/` is generator output and is gitignored (as is `openapi-generator-cli-*.jar`) — never hand-edit it, and don't treat it as source of truth; it may be stale relative to the templates. To change generated code, change the mustache template or the generator config.

## Commands

Local generation requires `openapi-generator-cli-7.15.0.jar` in the repo root ([download instructions](https://github.com/OpenAPITools/openapi-generator/tree/master?tab=readme-ov-file#13---download-jar)).

```bash
# Generate a client locally (mirrors the CI step)
java -jar openapi-generator-cli-7.15.0.jar generate \
  -i supporting-files/oas-input/petstore-api.yaml \
  --template-dir supporting-files/generator-templates/java/jersey3 \
  -c supporting-files/generator-configs/java-jersey3.yaml \
  -o ./generated-client

# Dump the mustache data model for all operations — the reference for what
# variables a template can use. Writes debugOperations JSON to stdout.
java -jar openapi-generator-cli-7.15.0.jar generate ... --global-property debugOperations=true
```

A previously captured dump lives at `supporting-files/generator-documents/petstore-debug-operations.json` (~3 MB) — grep it rather than reading it whole when you need to know whether a mustache variable exists.

Build/test the generated client (the generated `pom.xml` comes from `pom.mustache`; Java 17):

```bash
cd generated-client && mvn clean package     # single test: mvn test -Dtest=PetApiTest
cd generated-client && mvn spotless:apply    # google-java-format AOSP, configured in pom.mustache
```

## Architecture

**Pipeline layering** — `.github/workflows/generate-client-pipeline.yml` is the entry point (push to `main`/`feat/**`, or manual dispatch). It declares one job per API, each calling the reusable `generate-client-workflow.yml` with `openapi_spec_path`, `app_name` (→ Maven `artifactId`), `package_name` (→ Java package segment) and `package_version` (→ `artifactVersion`).

The reusable workflow has three jobs: `set-variables` (pins generator version 7.15.0, config path, template dir, output dir) → `generate-client` (runs `openapitools/openapi-generator-cli` in Docker, uploads the output as an artifact) → `package-and-deploy-client` (downloads the artifact, `mvn clean package deploy`). Deployment credentials come in as `-Dusername`/`-Dpassword`; the target repository is **hardcoded in `pom.mustache`'s `<distributionManagement>`**, not in the workflow.

**To add a new API**: drop the spec in `supporting-files/oas-input/`, then add a job to `generate-client-pipeline.yml`. Nothing else needs touching.

**Generator config** (`supporting-files/generator-configs/java-jersey3.yaml`) is shared by every API: `generatorName: java`, `library: jersey3`, `apiNameSuffix: api`, Jackson, Jakarta EE. Its `files:` block is the important part — it registers four *extra* per-API template outputs beyond the stock generator (`Client.java`, `MockProvider.java`, `MockConfiguration.java`, `ResponseExamples.java`, each suffixed onto the API class name, e.g. `PetApiClient.java`).

**The interface/implementation split is a customisation.** Upstream `api.mustache` emits a concrete class; here it emits `interface {{classname}}` (e.g. `PetApi`), and `api_client.mustache` emits `{{classname}}Client implements {{classname}}` with the actual Jersey invocation code. Anything consuming a generated client should depend on the interface so the mock can be substituted.

**The mock trio** (all custom templates, no upstream equivalent) is what makes this repo more than a vanilla generator run:

- `api_response_examples.mustache` → `{{classname}}ResponseExamples`: a Spring `@Component` that deserialises the response `example` payloads baked into the spec (`examples.0.example` in the mustache model) into typed fields named `{{operationId}}ResponseExample`, plus a `DEFAULT_HEADERS` map. Operations whose response schema carries no example produce nothing usable — examples in the OAS input are what drive mock fidelity.
- `api_mock.mustache` → `{{classname}}MockProvider`: `Mockito.spy` of the `Client`, with `doReturn(...)` stubs for every `{{operationId}}WithHttpInfo` returning a 200 `ApiResponse` built from the response examples.
- `api_mock_configuration.mustache` → `{{classname}}MockConfiguration`: Spring `@Configuration` exposing the spy as a `@Primary @Bean` typed as the *interface*. Bean and configuration names are prefixed with the camel-cased `artifactId`, so `app_name` in the pipeline must be unique across APIs sharing a Spring context.

Because these templates emit Mockito, Lombok and `spring-context` usage into `src/main`, `pom.mustache` declares those as **compile-scope** dependencies (see the "custom additions" block) — mocks ship inside the published client jar rather than a test jar.

**Template directory semantics**: `--template-dir` overrides only the files present in it; every other template (`licenseInfo.mustache`, `nullable_var_annotations.mustache`, …) falls back to the generator's built-in jersey3 set. The vendored `.mustache` files other than the four custom ones are copies of upstream 7.15.0 templates, so when bumping the generator version they need re-diffing against upstream.

**Generator version is pinned in three places** that must stay in sync: the root jar filename, `generator_version` in `generate-client-workflow.yml`, and the README's example commands.

## Known rough edges

- `package_version` is hardcoded per job in the pipeline; the intent (per the inline TODO) is to read it from the spec's `info.version`.
- `pom.mustache` carries a `TODO: replace with own artifact repository configuration` on `<distributionManagement>`.
