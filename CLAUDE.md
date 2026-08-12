# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Maven plugin that exposes the [Checkmarx AST CLI](https://github.com/Checkmarx/ast-cli) as a Maven goal, letting users run Checkmarx One scans from `mvn` against a project's `pom.xml`. Implemented as a single Mojo that forwards a free-form `-Darguments=...` string to `CxThinWrapper` from [ast-cli-java-wrapper](https://github.com/Checkmarx/ast-cli-java-wrapper) — all scan logic lives in the wrapper and the bundled CLI binary, not here.

**Status:** Active / maintained. Published to Maven Central as `com.checkmarx:ast-cli-maven-plugin`. Part of the Checkmarx One CI/CD integrations family (Gradle, Jenkins, JetBrains, VS Code, GitHub Action).

## Technology Stack

| Component | Details |
|-----------|---------|
| Language | Java 11 (`maven.compiler.source` / `target` = 11) |
| Build tool | Apache Maven, `packaging` = `maven-plugin` |
| Plugin API | `org.apache.maven:maven-plugin-api` 3.9.6 |
| Plugin annotations | `org.apache.maven.plugin-tools:maven-plugin-annotations` 3.11.0 (provided) |
| Maven project model | `org.apache.maven:maven-project` 2.2.1 |
| AST CLI bridge | `com.checkmarx.ast:ast-cli-java-wrapper` 2.4.25 |
| Publishing | `org.sonatype.central:central-publishing-maven-plugin` 0.8.0 (`autoPublish=true`) |
| Signing | `org.apache.maven.plugins:maven-gpg-plugin` 3.2.1 (loopback pinentry) |
| Sources jar | `org.apache.maven.plugins:maven-source-plugin` 3.3.1 |
| Test framework | None — no `src/test/` directory |
| Database | None |
| Web framework | None |

## Repository Structure

```
src/main/java/com/checkmarx/ast/cli/maven/plugin/
  └── RunCliMojo.java           Sole Mojo — implements the `run` goal
pom.xml                          Plugin POM (groupId com.checkmarx, packaging maven-plugin)
README.md                        User-facing install + usage docs
CODEOWNERS                       Review ownership
LICENSE                          Apache 2.0
logo.png                         Project logo
.github/
  ├── workflows/
  │   ├── release.yml                 Tag-triggered build + GPG sign + deploy to Maven Central
  │   ├── ast-scan.yaml               Checkmarx One self-scan (PR + push + daily 07:00 UTC)
  │   ├── scan-github-action.yml      Zizmor security scanner (PR + workflow_call)
  │   ├── manual-tag.yml              Manual tag creation (workflow_dispatch)
  │   ├── pr-automation.yml           Auto-add reviewers to PRs (pull_request_target)
  │   └── pr-linter.yml               Validate PR title and branch naming (pull_request)
  ├── ISSUE_TEMPLATE/
  └── pr-labeler.yml                PR labeler configuration (unused, workflow removed)
target/                             Maven build output (gitignored)
```

## Development Setup

**Prerequisites:** JDK 11, Apache Maven 3.x, git. GPG key + Sonatype Central credentials are required only for publishing (handled by CI).

```bash
git clone https://github.com/Checkmarx/ast-cli-maven-plugin.git
cd ast-cli-maven-plugin

mvn clean install                            # build + install to ~/.m2 (see README usage)
mvn package                                  # build the jar without installing
mvn versions:set -DnewVersion=X.Y.Z-SNAPSHOT # bump POM version locally (CI does this from the git tag)
mvn -DskipTests deploy                       # publish to Maven Central — CI only, needs secrets
```

**Invoking the plugin against another Maven project** (after `mvn install`):

```bash
mvn com.checkmarx:ast-cli-maven-plugin:<version>:run \
    "-Darguments=scan create --project-name my-project -s . --branch main" \
    -f pom.xml
```

The `-Darguments=...` string is passed verbatim to the AST CLI. See the [CLI Commands docs](https://checkmarx.atlassian.net/wiki/spaces/AST/pages/3039953091/CLI+Commands) for the argument grammar.

## API / Interfaces

The plugin exposes **one Mojo, one goal**.

**`run` goal** ([RunCliMojo.java](src/main/java/com/checkmarx/ast/cli/maven/plugin/RunCliMojo.java)):

```java
@Mojo(name = "run")
public class RunCliMojo extends AbstractMojo {
    @Parameter(property = "arguments")
    private String arguments;
    // execute() — see source
}
```

| Parameter | Property | Required | Description |
|-----------|----------|----------|-------------|
| `arguments` | `arguments` | yes | Raw AST CLI argument string. Empty/null → `MojoExecutionException("no arguments provided")`. |

Exception translation in `execute()`:

- `IOException` / `InterruptedException` → `MojoExecutionException` (infrastructure failure)
- `CxException` (CLI policy/threshold breach) → `MojoFailureException` (scan-policy failure)

## Architecture

A deliberately minimal adapter — Maven goal in, CLI subprocess out:

- [RunCliMojo.java](src/main/java/com/checkmarx/ast/cli/maven/plugin/RunCliMojo.java) — the sole Mojo. Validates `arguments` is non-empty, then calls `new CxThinWrapper().run(arguments)`.
- `CxThinWrapper` (external, from `ast-cli-java-wrapper`) — extracts the OS-appropriate AST CLI binary from the wrapper JAR to a temp directory and execs it with the given args. Output streams pass through unchanged.
- Exit semantics — Maven receives `MojoExecutionException` (build error) or `MojoFailureException` (policy failure); the calling CI distinguishes infrastructure breakage from "scan found vulnerabilities".

All scan logic, authentication, result formatting, and CLI flag parsing live in the AST CLI binary and `ast-cli-java-wrapper`. Bumping `ast-cli-java-wrapper` in `pom.xml` is how new CLI features reach plugin users.

## Project Rules (Invariants)

- **One Mojo, one goal — `run`.** The Gradle/JetBrains/VS Code plugins expose the CLI via a single passthrough surface. Adding new Mojos without integration-team coordination breaks user muscle memory across plugins.
- **Do not parse the `arguments` string in Java.** It is intentionally an opaque passthrough. Validating individual flags in the Mojo couples the plugin to CLI flag evolution and forces a plugin release every time the CLI gains a flag.
- **`MojoFailureException` vs `MojoExecutionException` is load-bearing.** `CxException` (policy/scan failure) → `MojoFailureException`; everything else → `MojoExecutionException`. CI branches on Maven's exit semantics — swapping these reclassifies "scan found vulnerabilities" as an infrastructure error.
- **Target Java 11.** `maven.compiler.source` / `target` are `11`. Bumping requires coordination with consumers (Jenkins shared libs, corporate Maven setups) who often lag.
- **`groupId` / `artifactId` are public coordinates** — `com.checkmarx:ast-cli-maven-plugin`. Never rename; user `pom.xml` files reference them by string.
- **`<version>dev</version>` is intentional on `main`.** CI rewrites it via `mvn versions:set -DnewVersion=<tag>` during release (see [release.yml:40](.github/workflows/release.yml#L40)). Do not bump the POM version manually on `main`.
- **CI release is tag-driven.** Pushing a git tag triggers [release.yml](.github/workflows/release.yml). Don't `mvn deploy` from a workstation.
- **GPG signing is mandatory for Maven Central.** `maven-gpg-plugin` runs in the `verify` phase. Disabling it in CI causes Sonatype Central to reject the artifact.

## Testing Strategy

The repository has **no unit/integration test sources** — there is no `src/test/` directory. Validation occurs through:

1. **Zizmor security validation** ([scan-github-action.yml](.github/workflows/scan-github-action.yml)) — runs on every PR to catch GitHub Actions security issues.
2. **Checkmarx One self-scan** ([ast-scan.yaml](.github/workflows/ast-scan.yaml)) — scans this repo's own source daily and on every PR with thresholds set to `=1` for high/medium/low across SCA/SAST/IaC.
3. **Release build** ([release.yml](.github/workflows/release.yml)) — `mvn deploy` proves the plugin compiles and packages correctly.
4. **Manual end-to-end runs** — integration testing against a target project's `pom.xml` by maintainers.

When adding behaviour to `RunCliMojo`, the expected pattern is `src/test/java/com/checkmarx/ast/cli/maven/plugin/` using `maven-plugin-testing-harness` — but that dependency is not currently in `pom.xml` and must be added.

## Known Issues / Limitations

- **No test suite.** Bugs in `RunCliMojo` are only caught at release time or by end users.
- **`arguments` is a single string, not a list.** Shell-level quoting is the caller's responsibility (`"-Darguments=... --project-name 'my project' ..."`). No JSON-array or multi-value form.
- **OS-dependent CLI extraction.** `CxThinWrapper` unpacks the bundled AST CLI binary to a temp directory. Hardened CI sandboxes (noexec `/tmp`) fail with cryptic `IOException`s — override `java.io.tmpdir`.
- **CLI version coupling.** Features available to users are fixed by the `ast-cli-java-wrapper` version. Users cannot independently upgrade the CLI without a new plugin release.
- **POM declares `<version>dev</version>` in source.** Local installs land as `com.checkmarx:ast-cli-maven-plugin:dev` until `mvn versions:set` runs. Released artifacts always carry the tag version.

## External Integrations

- **[ast-cli-java-wrapper](https://github.com/Checkmarx/ast-cli-java-wrapper)** — the sole runtime dependency that does real work. Bundles the AST CLI binary per OS and exposes `CxThinWrapper` / `CxException`. Bumping this version is the primary maintenance task.
- **[Checkmarx AST CLI](https://github.com/Checkmarx/ast-cli)** — the scanning tool, invoked as a subprocess by the wrapper.
- **Maven Central (Sonatype Central)** — publishing target. Configured via `central-publishing-maven-plugin` with `publishingServerId=central`, `autoPublish=true` ([pom.xml:55-65](pom.xml#L55-L65)).
- **Checkmarx One** — backend SaaS that receives scan submissions. Auth handled by the CLI's standard mechanisms (API key, OAuth2 client), not by the plugin.

## Deployment

N/A as a service — this is a library published to Maven Central, consumed via `<plugin>` declarations in user `pom.xml` files.

**Release flow:**

1. Maintainer pushes a git tag (e.g. `2.0.10`) to `main`, optionally via the `manual-tag.yml` workflow.
2. [release.yml](.github/workflows/release.yml) runs on tag push:
   - Sets up JDK 11 (Adopt) with OSSRH credentials and the GPG private key.
   - `mvn versions:set -DnewVersion=<tag>` rewrites the POM.
   - `mvn --batch-mode deploy -DskipTests` signs + uploads to Sonatype Central.
   - `step-security/action-gh-release` creates the GitHub Release with auto-generated notes.
3. Artifact appears at `https://repo.maven.apache.org/maven2/com/checkmarx/ast-cli-maven-plugin/<version>/` after Sonatype indexing (~15-30 min).

Required repo secrets: `PERSONAL_ACCESS_TOKEN`, `OSSRH_USERNAME`, `OSSRH_TOKEN`, `MAVEN_GPG_PRIVATE_KEY`, `MAVEN_GPG_PASSPHRASE`.

## Performance Considerations

- The Mojo is O(1) — one null/empty check and one delegated call. All wall-clock time is in the AST CLI subprocess (network + filesystem scan).
- `CxThinWrapper` extracts the CLI binary on every invocation. In multi-module Maven builds binding `run` to many modules, re-extraction cost compounds — bind the goal to a single aggregator module instead.
- No connection pooling or caching at the plugin layer — every `mvn ... :run` is an independent CLI invocation.

## Security & Access

- **No credentials handled in plugin code.** Auth is delegated to the AST CLI (env vars, `~/.checkmarx/`, or args). Do not add credential parameters to `RunCliMojo` — a second auth surface would diverge from the other Checkmarx integrations.
- **GPG-signed releases.** All Maven Central artifacts are signed; consumers verify with `gpg --verify ast-cli-maven-plugin-<version>.jar.asc`.
- **Self-scanned via Checkmarx One.** [ast-scan.yaml](.github/workflows/ast-scan.yaml) runs daily and on every PR with thresholds `=1` across SCA/SAST/IaC — any new finding fails the build.
- **Argument forwarding is by design.** `arguments` is a free-form string passed verbatim to the CLI. A `pom.xml` author can pass arbitrary CLI flags; that is the contract. Do not "sanitize" `arguments` in the Mojo.

## GitHub Actions Security

All workflows are scanned and validated using **Zizmor**, a GitHub Actions security linter:

- **[scan-github-action.yml](.github/workflows/scan-github-action.yml)** — runs on every PR and `workflow_call` trigger with pedantic persona
- Validates for: credential persistence (artipacked), template injection, concurrency limits, undocumented permissions, cache poisoning, dangerous triggers
- All workflows pass Zizmor validation with zero findings

**Key security practices enforced:**
- `persist-credentials: false` on all `actions/checkout` steps
- Template expressions (`${{ }}`) moved from `run:` blocks into `env:` variables
- Explicit `concurrency:` blocks to prevent workflow queue abuse
- Documented `permissions:` blocks at workflow and job levels
- Job `name:` fields for transparency (except required status checks)

## Logging

- The plugin **does not log directly**. It throws structured Maven exceptions; messages surface through Maven's logger.
- All scan output (progress, results, threshold reports) comes from the AST CLI subprocess via stdout/stderr. Maven streams it unchanged.
- For verbose CLI output, callers add `--debug` / `-v` to the `arguments` string — not to the `mvn` invocation.

## Coding Standards

- Java 11 source/target — no APIs newer than 11.
- Mojo classes live under `com.checkmarx.ast.cli.maven.plugin` and use `org.apache.maven.plugins.annotations.*` (not the legacy Javadoc `@parameter` form).
- Keep `RunCliMojo` small and delegating. New scan logic belongs in `ast-cli-java-wrapper` or the CLI, not here.
- Match the existing exception-translation pattern: `CxException` → `MojoFailureException`, everything else → `MojoExecutionException`.
- No checkstyle/spotless config — follow the existing 4-space indent and standard Java conventions in [RunCliMojo.java](src/main/java/com/checkmarx/ast/cli/maven/plugin/RunCliMojo.java).

## Debugging Steps

1. **Plugin not found by Maven:**
   ```bash
   mvn install                                # confirm local install
   mvn help:describe -Dplugin=com.checkmarx:ast-cli-maven-plugin
   ```
   Verify artifact id is `ast-cli-maven-plugin`, group `com.checkmarx`.

2. **`no arguments provided` error:** caller forgot `-Darguments=...` or passed an empty string. Rejected at [RunCliMojo.java:21](src/main/java/com/checkmarx/ast/cli/maven/plugin/RunCliMojo.java#L21).

3. **`MojoFailureException` vs `MojoExecutionException`:**
   - `Failure` → the scan ran and the CLI reported a policy/threshold breach (`CxException`). Inspect CLI output for the violating finding.
   - `Execution` → something broke before/around the scan (IO, interrupt, missing binary). Inspect the stack trace.

4. **CLI binary fails to start** (e.g. `Permission denied`, `Cannot run program`): noexec temp dir or missing execute bit on the extracted binary. Override:
   ```bash
   mvn ... -Djava.io.tmpdir=/var/tmp
   ```

5. **Release stuck / not on Maven Central:** check the [release.yml](.github/workflows/release.yml) run logs. Common causes: expired/invalid GPG key, missing `OSSRH_TOKEN`, Sonatype Central metadata rejection. After successful upload, indexing takes 15-30 minutes.

6. **Local install shows version `dev`:** expected — `<version>dev</version>` is the source-tree value. Run `mvn versions:set -DnewVersion=X.Y.Z-SNAPSHOT` before `mvn install` for a stable local coordinate.
