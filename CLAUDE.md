# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

There is **no application code here**. This repo is a *generator configuration* project: OpenAPI specs + a customised OpenAPI Generator template set + GitHub Actions workflows that produce Java (Jersey3) API clients *and Mockito/Spring mock providers*, then publish them as Maven artifacts to GitHub Packages.

Generator output goes to `target/generated-client/` and is gitignored (as are `generated-client/` from older runs and `openapi-generator-cli-*.jar`) — never hand-edit it, and don't treat it as source of truth; it may be stale relative to the templates. To change generated code, change the mustache template or the generator config.

## Commands

Local generation requires `openapi-generator-cli-7.15.0.jar` in the repo root ([download instructions](https://github.com/OpenAPITools/openapi-generator/tree/master?tab=readme-ov-file#13---download-jar)). **Always run from the repo root** — the generator configs set `templateDir` as a path relative to the working directory.

```bash
# Generate a client locally (mirrors the CI step)
java -jar openapi-generator-cli-7.15.0.jar generate \
  -i supporting-files/oas-input/petstore-api.yaml \
  -c supporting-files/generator-configs/java-jersey3.yaml \
  -o ./target/generated-client

# Dump the mustache data model for all operations — the reference for what
# variables a template can use. Writes debugOperations JSON to stdout.
java -jar openapi-generator-cli-7.15.0.jar generate ... --global-property debugOperations=true
```

A previously captured dump lives at `supporting-files/generator-documents/petstore-debug-operations.json` (~3 MB) — grep it rather than reading it whole when you need to know whether a mustache variable exists.

To reproduce CI exactly, run it through Docker. **`-w /local` is mandatory**: the image sets no `WorkingDir`, so the container cwd is `/` and the relative `templateDir` in the config would resolve outside the mount — silently producing nothing for configs without `templateDir`, and failing with `Template directory ... does not exist` for those with it.

```bash
docker run --rm -v "$PWD:/local" -w /local \
  openapitools/openapi-generator-cli:v7.15.0 generate \
  -i /local/supporting-files/oas-input/petstore-api.yaml \
  -c /local/supporting-files/generator-configs/java-jersey3.yaml \
  -o /local/target/generated-client
```

Build/test the generated client (the generated `pom.xml` comes from `pom.mustache`; Java 17):

```bash
cd target/generated-client && mvn clean package   # single test: mvn test -Dtest=PetApiTest
cd target/generated-client && mvn spotless:apply  # google-java-format AOSP, configured in pom.mustache
```

## Architecture

**Pipeline layering** — `.github/workflows/generate-client-pipeline.yml` is the entry point (push to `main`/`feat/**`, or manual dispatch). It declares one job per API, each calling the reusable `generate-client-workflow.yml` with `openapi_spec_path`, `generator_config_file`, `app_name` (→ Maven `artifactId`), `package_name` (→ Java package segment) and `package_version` (→ `artifactVersion`).

The reusable workflow has two jobs: `generate-client` (runs `openapitools/openapi-generator-cli` in Docker, uploads the output as an artifact) → `package-and-deploy-client` (downloads the artifact, then `mvn clean deploy`, or just `mvn clean package` when the `deploy` input is `false`). Generator version and output directory are workflow-level `env` values.

**Publishing.** Auth relies on a default nobody sets explicitly: `setup-java` always writes `~/.m2/settings.xml` containing a server with id `github`, whose credentials interpolate the `GITHUB_ACTOR` and `GITHUB_TOKEN` *environment variables*. `GITHUB_ACTOR` is always present; the deploy step sets `GITHUB_TOKEN` from `secrets.GITHUB_TOKEN`. The deploy target is passed as `-DaltDeploymentRepository=github::https://maven.pkg.github.com/${{ github.repository }}` — the `github` prefix must match that settings.xml server id. Nothing is hardcoded in `pom.mustache`, so poms generated from the *stock* templates (which have no `<distributionManagement>`) can also be deployed.

Publishing needs `packages: write` on the `GITHUB_TOKEN`. A called workflow's token can only be equal to or more restrictive than its caller's, so the grant appears in **both** `generate-client-pipeline.yml` (workflow level) and the `package-and-deploy-client` job. Removing either one yields a 401 at deploy time.

`app_name` must be unique per job: it becomes both the Maven `artifactId` and the upload/download artifact name, and `upload-artifact@v4` artifacts are immutable — two jobs uploading the same name in one run fail with a 409 conflict.

**Path ownership is deliberate.** The output directory lives in the workflow (`env.output_dir`, passed as `-o`) because the upload step has to know where the code landed. `templateDir` lives in the *config file* because it must vary per config — that is what distinguishes the two configs, and it cannot be hoisted to a shared CLI flag. Note that `templateDir` in a config is existence-checked inside `CodegenConfigurator.fromFile`, *before* CLI flags are applied, so a bad relative path there is fatal even if `--template-dir` is also passed.

**To add a new API**: drop the spec in `supporting-files/oas-input/`, then add a job to `generate-client-pipeline.yml` pointing at a generator config. Nothing else needs touching.

**Generator configs** live in `supporting-files/generator-configs/` and are selected per job. Both share `generatorName: java`, `library: jersey3`, `apiNameSuffix: api`, Jackson, Jakarta EE:

- `java-jersey3.yaml` — the real one. Sets `templateDir` to the custom template set, and its `files:` block registers four *extra* per-API template outputs beyond the stock generator (`Client.java`, `MockProvider.java`, `MockConfiguration.java`, `ResponseExamples.java`, each suffixed onto the API class name, e.g. `PetApiClient.java`).
- `java-jersey3-no-template.yaml` — no `templateDir`, no `files:`. Produces a stock client (a concrete `PetApi` class, no mocks) for comparison, built under `artifactId=petstore-no-template` with `deploy: false` so it is compiled but never published.

Both petstore variants generate into the *same* Java package, so they are drop-in alternatives — don't put both jars on one classpath, the API types collide on fully-qualified name.

**The interface/implementation split is a customisation.** Upstream `api.mustache` emits a concrete class; here it emits `interface {{classname}}` (e.g. `PetApi`), and `api_client.mustache` emits `{{classname}}Client implements {{classname}}` with the actual Jersey invocation code. Anything consuming a generated client should depend on the interface so the mock can be substituted.

**The mock trio** (all custom templates, no upstream equivalent) is what makes this repo more than a vanilla generator run:

- `api_response_examples.mustache` → `{{classname}}ResponseExamples`: a Spring `@Component` that deserialises the response `example` payloads baked into the spec (`examples.0.example` in the mustache model) into typed fields named `{{operationId}}ResponseExample`, plus a `DEFAULT_HEADERS` map. Operations whose response schema carries no example produce nothing usable — examples in the OAS input are what drive mock fidelity.
- `api_mock.mustache` → `{{classname}}MockProvider`: `Mockito.spy` of the `Client`, with `doReturn(...)` stubs for every `{{operationId}}WithHttpInfo` returning a 200 `ApiResponse` built from the response examples.
- `api_mock_configuration.mustache` → `{{classname}}MockConfiguration`: Spring `@Configuration` exposing the spy as a `@Primary @Bean` typed as the *interface*. Bean and configuration names are prefixed with the camel-cased `artifactId`, so `app_name` in the pipeline must be unique across APIs sharing a Spring context.

Because these templates emit Mockito, Lombok and `spring-context` usage into `src/main`, `pom.mustache` declares those as **compile-scope** dependencies (see the "custom additions" block) — mocks ship inside the published client jar rather than a test jar.

**Template directory semantics**: `--template-dir` overrides only the files present in it; every other template (`licenseInfo.mustache`, `nullable_var_annotations.mustache`, …) falls back to the generator's built-in jersey3 set. The vendored `.mustache` files other than the four custom ones are copies of upstream 7.15.0 templates, so when bumping the generator version they need re-diffing against upstream.

**Generator version is pinned in three places** that must stay in sync: the root jar filename, `generator_version` in `generate-client-workflow.yml`, and the README's example commands.

## Known rough edges

- `package_version` is hardcoded per job in the pipeline; the intent (per the inline TODO) is to read it from the spec's `info.version`. It is suffixed with `github.sha`, so every push publishes a new immutable release version.
