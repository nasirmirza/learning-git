# Shared Libraries, Parent POM and BOM for Services A, B, C

Nov 22, 2026 · @Nas Mir

## At a glance

**Put every version decision in two platform repos, `acme-parent` (Java, Spring Boot, third-party versions) and `acme-bom` (your own libraries), and give each service exactly one number to change: its `acme-service-parent` version.** Upgrading Java 17 → 25 and Spring Boot 3.5 → 4.1 then becomes a sequence of small releases, top to bottom in the picture, with each service moving when its own team is ready.

&#91;embedded content: seven repos · release order top to bottom\]

Read the arrows as "gets its versions from". Libraries build on `acme-parent` only, never on the BOM that lists them, which is what keeps a Boot major upgrade free of release cycles. The rest of this guide explains each box, then walks through every kind of change: library updates, CVEs, Boot patches and minors, and the move to Java 25 + Boot 4.1. The example code for all seven repos is in `acme-platform-repos.zip`.

## Why two parents: `acme-parent` and `acme-service-parent`

**Libraries and services need different things from a parent POM, so each gets its own: `acme-parent` holds what every Acme project shares, and `acme-service-parent` adds the two things only a deployable service needs.**

```text
acme-commons  ──►  acme-parent ──► spring-boot-starter-parent
service-a  ──►  acme-service-parent  ──►  acme-parent ──► spring-boot-starter-parent
                (+ Acme library versions,
                 + container image)
```

A library stops one level higher than a service.

|  | `acme-parent` | `acme-service-parent` |
| --- | --- | --- |
| Used by | Libraries (and services, through `acme-service-parent`) | Services only |
| Spring Boot and Java version | Defines them | Inherits them |
| Third-party versions, CVE overrides, enforcer rules | Defines them | Inherits them |
| Versions of your own libraries | No, to avoid a release cycle | Yes, by importing `acme-bom` |
| Executable jar and container image | No, libraries must stay plain jars | Yes, `spring-boot-maven-plugin` switched on |

### Why not one parent for everything?

Picture a single `acme-parent` that also imports `acme-bom` and switches on the Boot plugin, used by both libraries and services. Two things break.

1. **The libraries would come out as the wrong kind of jar.** The Boot plugin repackages a project into an executable "fat" jar with its own launcher and every dependency nested inside. That's right for `service-a`, which runs in a container. It's wrong for `acme-commons`, which services add as a dependency: classes inside a fat jar can't be used as a normal library.
2. **A Spring Boot major upgrade would go in a circle.** To build `acme-commons` 2.0.0 on Boot 4.1 you need parent 2.0.0 released first. But a parent 2.0.0 that lists library versions should list `acme-commons` 2.0.0, which can't exist until it has been built.

With the split, nothing points backwards: release `acme-parent` 2.0.0 (it doesn't mention your libraries), build the libraries on it, then release `acme-bom` and `acme-service-parent` 2.0.0 listing them, then bump the services.

### Why not skip `acme-service-parent` and import the BOM in each service?

That works too. Each service would extend `acme-parent` and import `acme-bom` itself:

```xml
<parent> acme-parent 2.0.0 </parent>
<dependencyManagement>
  import acme-bom 2.0.0
</dependencyManagement>
<build>
  spring-boot-maven-plugin + container image configuration
</build>
```

But then each of your three services carries two platform versions that must stay matched, plus its own copy of the plugin and image setup. `acme-service-parent` bundles all of that, so a service's whole platform setup is one line, and bumping it upgrades Java, Spring Boot and every shared library together:

```xml
<parent> acme-service-parent 2.0.0 </parent>
```

Spring Boot's own build used the same idea when it was built with Maven: a public BOM (`spring-boot-dependencies`) for what consumers get, kept separate from an internal parent (`spring-boot-parent`) for what the project itself needs ([Spring Boot wiki: Maven POM Files](https://github.com/spring-projects/spring-boot/wiki/Maven-POM-Files)).

## The seven repositories

**Two platform repos own every version, two library repos own shared code, and the three service repos own nothing but business code.** The split follows one rule: whatever changes for a different reason, or at a different pace, gets its own repo.

| Repo | Publishes | Decides | Changes when |
| --- | --- | --- | --- |
| `acme-parent` | `acme-parent` | Spring Boot line, Java level, plugin conventions, enforcer rules, third-party versions Boot doesn't manage, CVE overrides | Boot patch/minor/major, Java upgrade, CVE, new build rule |
| `acme-commons` | `acme-commons` jar | Shared runtime code (here: correlation IDs and problem responses, auto-configured) | Feature or fix in the shared code; every platform major |
| `acme-test-support` | `acme-test-support` jar | Shared test code (here: `@AcmeServiceTest`, `AcmeResultMatchers`) | Feature or fix; Spring Boot test-API changes |
| `acme-bom` | `acme-bom` + `acme-service-parent` | Which versions of Acme's own libraries go together; how a service is packaged as a container | A library is released; `acme-parent` is released |
| `service-a`, `service-b`, `service-c` | Container images | Business code; their own release version | Features; a platform bump PR |

### Why `acme-parent` and `acme-bom` are separate

If one POM held both the Boot version and the library versions, you couldn't do a Boot major upgrade: the libraries must be rebuilt on the new Boot line *before* anything can list their new versions, but they would need that very POM to build. Splitting it breaks the cycle:

- Libraries build on **`acme-parent`**, which knows nothing about Acme libraries.
- Services build on **`acme-service-parent`**, which is `acme-parent` + `acme-bom`.

### Why `acme-service-parent` lives in the BOM repo

It exists only to combine `acme-parent` with `acme-bom`, so it is released with the BOM, at the same version. Services then have **one** number to bump. Without it, each service would carry two platform versions (parent and BOM import) that could drift apart.

### What deliberately has no repo

- **No Spring Cloud BOM, database drivers, Testcontainers.** Add them to `acme-parent` the same way as `springdoc.version` when a service needs them.
- **No aggregate "build everything" repo.** The repos release independently; `build-all.ps1` in the example folder is a local convenience, not part of the design.

## Versioning and the compatibility matrix

**Platform artifacts use semantic versions whose MAJOR number is the "platform line": a Java baseline plus a Spring Boot generation.** A service team can tell the risk of a bump from the number alone.

| Bump | In `acme-parent` / `acme-bom` it means | Service team effort |
| --- | --- | --- |
| PATCH `1.0.0 → 1.0.1` | Spring Boot patch, CVE override, library bug fix | None: merge the bot PR when CI is green (can auto-merge) |
| MINOR `1.0.x → 1.1.0` | Spring Boot minor (e.g. 4.1 → 4.2), new library feature, new enforcer rule | Read the release notes; usually no code change |
| MAJOR `1.x → 2.0.0` | New Java baseline and/or Spring Boot major; library majors | Planned migration with code changes |

Shared libraries follow the same rule: their MAJOR moves with the platform line, because a jar compiled for Java 25 and Spring Framework 7 can't run on a Java 17 / Boot 3.5 service.

### The matrix (keep this table in the `acme-bom` README)

| Platform line | `acme-parent` | `acme-bom` / `acme-service-parent` | Spring Boot | Java | `acme-commons` | `acme-test-support` | Status |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 2 | 2.0.0 | 2.0.0 | 4.1.2 | 25 | 2.0.0 | 2.0.0 | Current |
| 1 | 1.0.0 | 1.0.0 | 3.5.16 | 17 | 1.0.0 | 1.0.0 | Maintenance until all services move; Boot 3.5 open-source support ended 30 Jun 2026 |

`acme-parent` and `acme-bom` share a major but not necessarily a minor or patch. A CVE fix ships `acme-parent 2.0.1`, then `acme-bom 2.0.1` re-points `acme-service-parent` at it. Keeping their numbers equal is a convenience, not a rule.

### Spring Boot dates that drive the plan

- Boot **3.5**: last open-source patch was 3.5.16; open-source support ended 30 June 2026. Your services are already past it, which makes the move urgent.
- Boot **4.1**: current patch 4.1.2; open-source support until 31 July 2027. It supports Java 17 through 27, so Java 25 is fine.
- Boot **4.2** was scheduled for 19 November 2026. Once you're on 4.1, moving to 4.2 is a MINOR platform release (section on Boot upgrades). Check whether it has shipped before you start; starting directly on 4.2 may save one round.

Sources: [Spring Boot 3.5.16 release](https://spring.io/blog/2026/06/25/spring-boot-3-5-16-available-now/), [Boot support calendar](https://ecorpit.com/spring-boot-4-1-end-of-support-2027/).

## Inside each repo

**Each POM below has one job; the excerpts are from the example code at platform line 2 (Java 25, Boot 4.1.2).** The full files carry comments on every decision.

### `acme-parent` — conventions and third-party versions

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>4.1.2</version>            <!-- (1) THE Spring Boot version -->
  <relativePath/>
</parent>
<groupId>com.acme.platform</groupId>
<artifactId>acme-parent</artifactId>
<version>2.0.0</version>
<packaging>pom</packaging>

<properties>
  <java.version>25</java.version>      <!-- (2) THE Java level -->
  <springdoc.version>3.0.1</springdoc.version>  <!-- (3) third-party, not managed by Boot -->
  <!-- (4) CVE overrides of Boot-managed libraries go here as Boot version properties -->
</properties>
```

- **(1)** Extending Boot's starter parent gives every library and service Boot's tested versions *and* its plugin setup, and keeps Boot's version properties working for CVE overrides.
- **(2)** `java.version` feeds three things: `maven.compiler.release` (via Boot's parent), the enforcer's `requireJavaVersion [25,)`, and the container JVM (via `acme-service-parent`).
- **(3)** springdoc is a good example of a library Boot doesn't manage but which must move with Boot: 2.8.x works with Boot 3, 3.0.x with Boot 4. Its version lives here so it changes in the same release as Boot.
- The **enforcer** execution runs in every repo: right JDK, Maven ≥ 3.9.6, no duplicate or dynamic versions, no JUnit 4 or Log4j 1.

### `acme-commons` and `acme-test-support` — libraries

```xml
<parent>
  <groupId>com.acme.platform</groupId>
  <artifactId>acme-parent</artifactId>   <!-- NOT acme-service-parent -->
  <version>2.0.0</version>
  <relativePath/>
</parent>
<artifactId>acme-commons</artifactId>
<version>2.0.0</version>

<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-webmvc</artifactId>
  <optional>true</optional>            <!-- compile against it, don't force it on consumers -->
</dependency>
<dependency>
  <groupId>jakarta.servlet</groupId>
  <artifactId>jakarta.servlet-api</artifactId>
  <scope>provided</scope>               <!-- the service's Tomcat supplies it -->
</dependency>
```

Rules every shared library follows:

1. **No versions for anything Spring Boot manages.** The library compiles against the same versions services run.
2. **Auto-configuration, not instructions.** `acme-commons` registers its beans through `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`, guarded by `@ConditionalOnWebApplication` and `@ConditionalOnMissingBean`, so adding the jar is all a service does, and a service can still replace any bean.
3. **Stick to Spring Framework APIs where you can.** `acme-commons` uses only Framework and SLF4J classes, so its code didn't change at all between Boot 3.5 and 4.1; only the rebuild and major version did.
4. **A library that depends on another Acme library pins it in a property** (`acme-test-support` → `acme-commons.version`). In a service, `acme-bom`'s managed version overrides that pin, so all Acme jars stay aligned even transitively.
5. **The test library is compile-scoped internally and test-scoped for consumers.** `acme-test-support` depends on `spring-boot-starter-test` at compile scope; `acme-bom` manages `acme-test-support` with `<scope>test</scope>`, so it can never reach a container image.

### `acme-bom` — Acme's own versions, nothing else

```xml
<artifactId>acme-bom</artifactId>
<version>2.0.0</version>
<packaging>pom</packaging>           <!-- no <parent>, no Spring Boot import -->
<properties>
  <acme-commons.version>2.0.0</acme-commons.version>
  <acme-test-support.version>2.0.0</acme-test-support.version>
</properties>
<dependencyManagement>
  <dependencies>
    <dependency> acme-commons ${acme-commons.version} </dependency>
    <dependency> acme-test-support ${acme-test-support.version} <scope>test</scope> </dependency>
  </dependencies>
</dependencyManagement>
```

- **No `<parent>`.** Anything a BOM inherits is copied into everyone who imports it.
- **No Spring Boot import, no third-party pins.** Services already inherit Boot through `acme-parent`, and inherited versions beat imported ones, so a third-party pin here would be silently ignored for every service (the effect shown in the Parent POM & BOM deep dive tab of your Maven guide). Third-party versions belong in `acme-parent`.

### `acme-service-parent` — the services' single dial

```xml
<parent>
  <groupId>com.acme.platform</groupId>
  <artifactId>acme-parent</artifactId>
  <version>2.0.0</version>
</parent>
<artifactId>acme-service-parent</artifactId>
<version>2.0.0</version>
<properties>
  <acme-platform.version>2.0.0</acme-platform.version>
</properties>
<dependencyManagement>   import acme-bom ${acme-platform.version}   </dependencyManagement>
<build><plugins>
  <plugin> spring-boot-maven-plugin
    <image>
      <name>acme/${project.artifactId}:${project.version}</name>
      <env><BP_JVM_VERSION>${java.version}</BP_JVM_VERSION></env>
    </image>
  </plugin>
</plugins></build>
```

- **The trap the checks caught.** The first draft imported the BOM with `${project.version}`. Properties in a parent are interpolated in the *child*, so in `service-a` that became `1.0.0-SNAPSHOT` (the service's own version) and the build failed with "Non-resolvable import POM acme-bom:1.0.0-SNAPSHOT". Never use `${project.version}` in a parent to mean the parent's version; use a property of its own.
- **Container image follows Java.** `mvn spring-boot:build-image` builds an OCI image whose JVM version comes from `java.version`, so a Java upgrade can't forget the base image. If you build with Dockerfiles instead, the base image tag (`eclipse-temurin:25-jre`) is one more place to update.
- Services, not libraries, get the Boot plugin switched on (executable jar, image).

### `service-a` — what a service declares

```xml
<parent>
  <groupId>com.acme.platform</groupId>
  <artifactId>acme-service-parent</artifactId>
  <version>2.0.0</version>      <!-- the ONLY platform version in the service -->
  <relativePath/>
</parent>
<groupId>com.acme.services</groupId>
<artifactId>service-a</artifactId>
<version>1.0.0-SNAPSHOT</version>  <!-- the service's own version -->

<dependencies>
  <dependency> org.springframework.boot : spring-boot-starter-webmvc </dependency>
  <dependency> org.springframework.boot : spring-boot-starter-actuator </dependency>
  <dependency> com.acme.platform : acme-commons </dependency>
  <dependency> org.springdoc : springdoc-openapi-starter-webmvc-ui </dependency>
  <dependency> com.acme.platform : acme-test-support <scope>test</scope> </dependency>
</dependencies>
```

Not one `<version>` in the dependencies. The inheritance chain that supplies them:

```text
service-a
└─ acme-service-parent 2.0.0      imports acme-bom 2.0.0 (Acme libraries), packaging, image
   └─ acme-parent 2.0.0           Java 25, enforcer, springdoc, CVE overrides
      └─ spring-boot-starter-parent 4.1.2
         └─ spring-boot-dependencies 4.1.2   (several hundred managed versions)
```

## Release order

**Every change flows the same way: `acme-parent` → libraries → `acme-bom` → services, and each step only starts once the previous one is released.** Nothing ever points backwards, so there is no cycle to untangle.

1. **`acme-parent`** — needs only Spring Boot from Maven Central.
2. **`acme-commons`** — needs `acme-parent`.
3. **`acme-test-support`** — needs `acme-parent` and `acme-commons` (it uses the correlation header constant).
4. **`acme-bom`** (with `acme-service-parent`) — lists the library versions released in steps 2–3 and extends the `acme-parent` from step 1.
5. **`service-a`, `service-b`, `service-c`** — in any order, independently, whenever each team merges its bump.

Not every change touches every step. Most changes enter partway along:

| Change | Starts at | Repos that release |
| --- | --- | --- |
| Bug fix in `acme-commons` | Step 2 | `acme-commons`, `acme-bom`, then services bump |
| CVE in a Boot-managed library | Step 1 | `acme-parent`, `acme-bom` (re-point), then services bump. Libraries needn't re-release |
| Spring Boot patch | Step 1 | Same as a CVE |
| Spring Boot minor | Step 1 | Same, plus libraries if their tests fail on the new minor |
| Java or Spring Boot major | Step 1 | All seven, in order |

### What "released" means

A tagged commit (`v2.0.1`) whose CI job ran `mvn deploy` to your artifact repository (Nexus or Artifactory). Released versions are immutable: never redeploy over one. A mistake is fixed by the next version, never by overwriting.

While you're developing across repos on one machine, `mvn install` puts SNAPSHOT builds in your local `~/.m2` so the next repo can see them. That's what `build-all.ps1` does. CI should never depend on SNAPSHOTs from another repo.

## Scenario: updating a shared library

**A library change reaches services only through a BOM release, so services never pick up a library version the platform hasn't listed.** How far it goes depends on whether the change breaks callers.

### Bug fix or backward-compatible feature (PATCH or MINOR)

Example: `acme-commons` should also accept the header `X-Request-Id`.

1. **`acme-commons`**: change the code and its tests on `main`. Set the version to 2.1.0 (new feature) or 2.0.1 (fix), tag `v2.1.0`, push; CI deploys.
2. **`acme-test-support`** (only if it needs the new feature): bump `<acme-commons.version>` to 2.1.0 and release it. Otherwise leave it: the BOM aligns `acme-commons` for services anyway.
3. **`acme-bom`**: set `<acme-commons.version>2.1.0</acme-commons.version>`, release `acme-bom` + `acme-service-parent` 2.1.0.
4. **Services**: Renovate opens "Update acme platform to 2.1.0" in each service. CI green → merge.

```bash
# in acme-commons
mvn versions:set -DnewVersion=2.1.0
git commit -am "Accept X-Request-Id" && git tag v2.1.0 && git push --follow-tags

# in acme-bom, after acme-commons 2.1.0 is deployed
mvn versions:set-property -Dproperty=acme-commons.version -DnewVersion=2.1.0
mvn versions:set -DnewVersion=2.1.0 -DprocessAllModules=true
mvn versions:set-property -Dproperty=acme-platform.version -DnewVersion=2.1.0
git commit -am "acme-commons 2.1.0" && git tag v2.1.0 && git push --follow-tags
```

### Breaking change (MAJOR)

Example: rename `ResourceNotFoundException` to `NotFoundException`.

1. Avoid it if you can. Add the new class, deprecate the old one, remove it at the next platform major. That turns a breaking change into a MINOR.
2. If you must break it: release `acme-commons 3.0.0`. Release a new `acme-bom` major that lists it, with release notes saying exactly what each service must change.
3. Services take the bump as a planned migration, not an auto-merge (the Renovate rules in the example only auto-merge PATCH).

### Testing a library change against a service before releasing it

```bash
cd acme-commons && mvn install          # 2.1.0-SNAPSHOT into ~/.m2
cd ../service-a
mvn verify -Dacme-commons.version=2.1.0-SNAPSHOT   # does NOT work: BOM properties can't be overridden
```

That last line is the BOM rule from your deep-dive tab: a property inside an imported BOM can't be overridden by the importer. Instead, add a temporary entry to the service's own `dependencyManagement`, which beats the import, and don't commit it:

```xml
<dependencyManagement><dependencies>
  <dependency>
    <groupId>com.acme.platform</groupId>
    <artifactId>acme-commons</artifactId>
    <version>2.1.0-SNAPSHOT</version>
  </dependency>
</dependencies></dependencyManagement>
```

## Scenario: a CVE in a third-party library

**The fix goes into `acme-parent` once, as a PATCH release, and reaches all three services through one bot PR each. No service edits a version.** Pick the path by where the vulnerable library's version comes from.

### Step 0: find out who brings it in

```bash
cd service-a
mvn dependency:tree -Dincludes=org.apache.tomcat.embed     # which path pulls it in
mvn help:effective-pom -Dverbose | findstr tomcat.version   # PowerShell/cmd; use grep on bash
```

| The vulnerable library is… | Fix in | How |
| --- | --- | --- |
| Managed by Spring Boot, and a Boot patch already fixes it | `acme-parent` | Bump the `spring-boot-starter-parent` version (e.g. 4.1.2 → 4.1.3). Best option: you get Boot's tested combination |
| Managed by Spring Boot, no Boot patch yet | `acme-parent` | Override Boot's version property, e.g. `<tomcat.version>`. Property names are in the Boot reference under "Version Properties" |
| Managed by Boot but has no property | `acme-parent` | Add an explicit `<dependencyManagement>` entry for that artifact |
| Not managed by Boot (e.g. springdoc) | `acme-parent` | Bump its property (`springdoc.version`) |
| Pulled in only by an Acme library | that library, then `acme-bom` | Fix the library's dependency, release a PATCH, list it in the BOM |

### The override, as committed in `acme-parent`

```xml
<properties>
  <!-- SEC-481: CVE-yyyy-nnnnn in Tomcat. Remove when spring-boot-starter-parent ships Tomcat >= x.y.z. -->
  <tomcat.version>x.y.z</tomcat.version>
</properties>
```

Every override needs a ticket and a removal condition in the comment. Otherwise overrides outlive the fix and later hold a library *back* when Boot moves past it.

### The rollout

1. `acme-parent` 2.0.0 → **2.0.1** with the override. Tag, CI deploys.
2. `acme-bom`: `acme-service-parent`'s `<parent>` → `acme-parent 2.0.1`; release **2.0.1**. (Library versions are unchanged.)
3. Renovate opens "Update acme platform to 2.0.1" in each service; PATCH rules auto-merge it when CI is green.
4. Deploy each service.
5. Verify in each service: `mvn dependency:tree -Dincludes=org.apache.tomcat.embed` shows the fixed version. Your scanner (OWASP Dependency-Check, Snyk, GitHub Dependabot alerts or a CycloneDX SBOM fed to Dependency-Track) should clear.

Do libraries need a re-release? No. A library's POM only *requests* versions; inside a service, `acme-parent`'s management decides what ships. Libraries matter only if their own tests or CI must be clean too, which they will be the next time they bump `acme-parent`.

### The emergency path

If one service must ship before the platform release, put the same property override directly in that service's POM with the same comment. Renovate's next platform PR is the reminder to delete it. This works because Boot's versions reach the service through inheritance, where a property set lower down wins.

## Scenario: Spring Boot patch and minor upgrades

**A Boot patch is a one-line platform PATCH release; a Boot minor is a platform MINOR release whose libraries are tested on the new minor first.** Plan for one minor every six months: Boot releases in May and November and supports each minor for about 12 months.

### Patch (4.1.2 → 4.1.3)

1. `acme-parent`: change the `spring-boot-starter-parent` version. Remove any CVE overrides that this patch makes unnecessary. Release **2.0.x+1**.
2. `acme-bom`: point `acme-service-parent` at the new `acme-parent`; release.
3. Services: Renovate PR, auto-merge on green.

### Minor (4.1 → 4.2)

1. **Read the release notes** for the new minor: deprecations removed, property renames, dependency upgrades. Spring Boot's wiki has one page per minor.
2. **Branch `acme-parent`**, change the Boot version, and update third-party versions tied to Boot (springdoc and the like). Build it locally with `mvn install`.
3. **Build both libraries against the local SNAPSHOT parent.** Fix and release any library whose tests fail. Libraries that pass needn't re-release.
4. **Release `acme-parent` 2.1.0**, then `acme-bom` 2.1.0 listing any new library versions.
5. **Canary**: merge the bump in one service first (pick the one with the best tests). Deploy it, watch it for a day.
6. **The rest**: Renovate PRs for MINOR are labelled `platform-upgrade` and need a human approval, not auto-merge.

### What can surprise you in a minor

- A transitive library jumps a minor (Hibernate, Jackson, Micrometer) and changes behaviour. Your service tests are the safety net, which is why `acme-test-support` gives every service the same baseline test setup.
- A property is renamed. Add `spring-boot-properties-migrator` (runtime scope) temporarily: at startup it logs every renamed property it finds.
- Your CVE overrides now *downgrade* something. Remove the overrides as step 2.

## Scenario: Java 17 + Boot 3.5 → Java 25 + Boot 4.1

**This is platform line 2: every repo releases a new major in the fixed order, while line 1 stays available so no service is forced to move on the same day.** In the example code, `git diff v1.0.0 v2.0.0` in each platform repo and `git diff on-platform-1 on-platform-2` in each service show exactly this migration.

### Phase 0: prepare on line 1 (no new major yet)

Each of these works on Boot 3.5 and makes the later diff smaller:

- [ ] Make sure every service is on the newest line-1 platform (Boot 3.5.16).
- [ ] Replace `@MockBean` / `@SpyBean` with `@MockitoBean` / `@MockitoSpyBean` (`org.springframework.test.context.bean.override.mockito`). They come from Spring Framework 6.2, which Boot 3.5 already uses; Boot 4 removes the old ones.
- [ ] Fix every deprecation warning: Boot 4 removes what 3.x deprecated.
- [ ] Find direct uses of Jackson (`ObjectMapper`, `@JsonComponent`, custom serializers). Boot 4 defaults to Jackson 3, whose packages move from `com.fasterxml.jackson.*` to `tools.jackson.*` (annotations stay in `com.fasterxml.jackson.annotation`). The example services don't touch Jackson, so they had nothing to change.
- [ ] Install JDK 25 on developer machines and CI runners.

### Phase 1: `acme-parent` 2.0.0

```diff
   <artifactId>spring-boot-starter-parent</artifactId>
-  <version>3.5.16</version>
+  <version>4.1.2</version>
   ...
-  <version>1.0.0</version>
+  <version>2.0.0</version>
   ...
-  <java.version>17</java.version>
+  <java.version>25</java.version>
-  <springdoc.version>2.8.12</springdoc.version>
+  <springdoc.version>3.0.1</springdoc.version>
```

Also drop every CVE override (Boot 4.1 has moved past them) and bump CI's JDK to 25. Release as `2.0.0-RC1` first if you want to shake out problems before calling it final.

### Phase 2: libraries 2.0.0

| Library | Change for line 2 | Why |
| --- | --- | --- |
| `acme-commons` | Parent → `acme-parent 2.0.0`, version → 2.0.0. **No code change.** | Uses only Spring Framework and SLF4J APIs, which didn't move. Major bump because the jar is now Java 25 bytecode built for Spring Framework 7 |
| `acme-test-support` | Parent and version as above; `acme-commons.version` → 2.0.0; add `spring-boot-starter-webmvc-test`; `AutoConfigureMockMvc` import moves to `org.springframework.boot.webmvc.test.autoconfigure` | Boot 4 split test support into per-technology modules |

Keep a `1.x` branch in each library for line-1 fixes until the last service leaves line 1.

### Phase 3: `acme-bom` 2.0.0

Library versions → 2.0.0; `acme-service-parent`'s parent → `acme-parent 2.0.0`; release. Line 2 now exists.

### Phase 4: services, one at a time

The whole diff for `service-a` in the example:

```diff
   <artifactId>acme-service-parent</artifactId>
-  <version>1.0.0</version>
+  <version>2.0.0</version>
 ...
-  <artifactId>spring-boot-starter-web</artifactId>
+  <artifactId>spring-boot-starter-webmvc</artifactId>
 ...
-import org.springframework.boot.test.mock.mockito.MockBean;
+import org.springframework.test.context.bean.override.mockito.MockitoBean;
-    @MockBean
+    @MockitoBean
```

Plus the JDK in CI. With Phase 0 done, only the parent version and the starter name are left.

For real services, which do more than the examples, also expect:

- **Starter renames.** Boot 4 renamed several starters (`web` → `webmvc`, `aop` → `aspectj`, `oauth2-resource-server` → `security-oauth2-resource-server`) and added per-technology test starters. The `-classic` starters are a stop-gap that restores the Boot 3 layout.
- **Jackson 3** wherever you use Jackson directly.
- **Property renames**: run once with `spring-boot-properties-migrator`.
- **OpenRewrite** has a Boot 3 → 4 recipe that does much of the mechanical work; run it on a branch and review the diff.

Rollout: canary service first, then the others. Each deploys on its own schedule; line 1 and line 2 services talk HTTP to each other just fine, because the platform line is a build-time choice, not a wire protocol.

### Do Java and Boot have to move together?

No. Boot 4.1 supports Java 17 through 27, so a more cautious path is line 2 = Boot 4.1 on Java 17, then line 3 = Java 25 alone. That's two smaller majors instead of one larger one. For three services, one major with a canary is usually the better trade.

Sources: [Spring Boot 4.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide), [Modularizing Spring Boot](https://spring.io/blog/2025/10/28/modularizing-spring-boot), [Spring Boot 4.1.0 release](https://spring.io/blog/2026/06/10/spring-boot-4/).

## Other scenarios

**Each of these is a variation of the same flow; the table says where it starts and what to watch.**

| Scenario | What to do | Watch out for |
| --- | --- | --- |
| **New service D** | Copy a service repo (or start from start.spring.io), replace `<parent>` with the current `acme-service-parent`, delete every `<version>` from dependencies, add `renovate.json` and CI | Starters from start.spring.io come with a Boot parent and versions: remove both |
| **New shared library** | New repo with `acme-parent` as parent, auto-configuration for anything it wires up. Release 1.0.0, then add it to `acme-bom` | Never give it `acme-service-parent` as parent: that creates the release cycle |
| **A service needs a third-party library no other service uses** | Add its version to `acme-parent` anyway (property + `dependencyManagement`), unless Boot already manages it | Versions hidden in one service's POM get forgotten at the next Boot major |
| **One service lags on line 1** | It keeps `acme-service-parent 1.x`. Fixes for it come from `1.x` branches of `acme-parent`, the libraries and `acme-bom` | Set a deadline: line 1 is on Boot 3.5, whose open-source support has ended. Backporting doubles every fix |
| **Hotfix on line 1** (CVE while one service is still there) | `acme-parent` branch `1.x` → 1.0.1 with the override; `acme-bom` branch `1.x` → 1.0.1. The lagging service bumps | Cherry-pick the same fix to line 2 too |
| **Rollback after a bad platform release** | Revert the service's parent bump and redeploy the previous image. Then release a fixed platform PATCH | Never delete or overwrite a released version in Nexus; just stop pointing at it |
| **Library bug only one service hits** | Fix in the library, release PATCH, release `acme-bom`; every service gets it | Don't let that service pin a private library version: other services would then run untested combinations |
| **Two libraries need different versions of a third-party jar** | The platform decides one version in `acme-parent`; fix whichever library can't run on it | `mvn dependency:tree -Dverbose` shows the conflict; the enforcer's `dependencyConvergence` rule can catch it in CI |
| **Spring Cloud or another vendor BOM Boot doesn't manage** | Import it in `acme-parent`'s `dependencyManagement`, version in a property | Spring Cloud release trains support specific Boot minors: bump both in the same release |
| **Newer version of a family Boot already manages** (Testcontainers, Jackson) | Override Boot's version property in `acme-parent`, e.g. `<testcontainers.version>` | Importing that vendor's BOM in `acme-parent` does nothing: Boot's inherited versions beat imports |

## Automation and guardrails

**Four pieces make the setup run itself: an artifact repository, CI per repo, a bot that opens version bumps, and build rules that fail on drift.** Each repo in the example already has the CI and bot files.

### 1. Artifact repository (Nexus or Artifactory)

- A **releases** repository (immutable) and a **snapshots** repository; the URLs go in `distributionManagement` (the example uses placeholders at `nexus.acme.example`).
- A **group/proxy** that mirrors Maven Central, so builds don't depend on Central's uptime and you can block known-bad versions in one place.
- Every developer and CI runner points at it in `~/.m2/settings.xml`:

```xml
<settings>
  <mirrors>
    <mirror>
      <id>acme-group</id>
      <mirrorOf>*</mirrorOf>
      <url>https://nexus.acme.example/repository/maven-public/</url>
    </mirror>
  </mirrors>
  <servers>
    <server><id>acme-releases</id><username>${env.NEXUS_USER}</username><password>${env.NEXUS_PASSWORD}</password></server>
  </servers>
</settings>
```

### 2. CI in every repo (`.github/workflows/ci.yml` in the example)

| Repo kind | On pull request | On tag `vX.Y.Z` | On `main` |
| --- | --- | --- | --- |
| Platform and libraries | `mvn -B verify` | `mvn -B deploy` to Nexus | `mvn -B verify` |
| Services | `mvn -B verify` | — | `spring-boot:build-image`, push, deploy |

The JDK version in the workflow (`java-version: '25'`) is the one place outside `acme-parent` that names Java. The enforcer rule catches a mismatch: a JDK 17 runner fails at `requireJavaVersion [25,)` with a clear message.

### 3. Renovate (`renovate.json` in the example)

- Watches `com.acme.platform:*` in every service and library and opens one grouped PR per platform release.
- **PATCH** bumps auto-merge when CI is green: that's how a CVE fix reaches all services within hours.
- **MINOR/MAJOR** bumps get the `platform-upgrade` label and need a human.
- Point Renovate at your Nexus too (a `hostRules` entry), or it can't see your internal versions.
- Dependabot can do the same with `groups` in `.github/dependabot.yml` if you prefer GitHub's built-in bot.

### 4. Build rules (in `acme-parent`)

| Rule | Catches |
| --- | --- |
| `requireJavaVersion [${java.version},)` | Wrong JDK on a laptop or runner |
| `requireMavenVersion [3.9.6,)` | Old Maven with different resolution behaviour |
| `banDuplicatePomDependencyVersions` | The same dependency declared twice |
| `banDynamicVersions` | Ranges, `LATEST`, `RELEASE` |
| `bannedDependencies` | JUnit 4, Log4j 1; add anything your security team bans |
| `dependencyConvergence` (add once the services are clean) | Two paths asking for different versions of an unmanaged jar |

### 5. Knowing where each service stands

A platform owner should be able to answer "which services are still on line 1?" in seconds. Options, from lightest:

- Renovate's **Dependency Dashboard** issue in each repo lists pending platform bumps.
- A scheduled job that reads `<parent><version>` from each service repo and writes a table.
- Generate an SBOM per image (CycloneDX Maven plugin) and feed it to Dependency-Track: it shows versions *and* known CVEs per deployed service.

## Running the example code

**The download `acme-platform-repos.zip` holds seven folders, each a git repository with two tagged commits: platform line 1 (today) and line 2 (after the upgrade).** `build-all.ps1` installs them into your local `~/.m2` in release order.

| Repo | Line 1 tag | Line 2 tag |
| --- | --- | --- |
| `acme-parent`, `acme-commons`, `acme-test-support`, `acme-bom` | `v1.0.0` | `v2.0.0` |
| `service-a`, `service-b`, `service-c` | `on-platform-1` | `on-platform-2` |

```powershell
# PowerShell, from the unzipped acme-repos folder
.\build-all.ps1 -Line 1        # needs JDK 17+ as JAVA_HOME
.\build-all.ps1 -Line 2        # needs JDK 25+
cd service-a; mvn spring-boot:run
# then: curl -i http://localhost:8080/orders/42     (watch the X-Correlation-Id header)
#       curl -i http://localhost:8080/orders/1      (a 404 problem response from acme-commons)
```

### How it was checked

The sandbox I built this in can't reach Maven Central, so **the Java code has not been compiled or run against real Spring Boot yet**. What was checked: every POM on both lines was resolved with Maven 3.9.11's own model builder, against stand-in Spring Boot POMs with the same structure as the real ones. All nine POMs per line resolve with no missing versions, the expected Java level (17 / 25) and the expected versions for every dependency. That check is also what caught the `${project.version}` trap described in "Inside each repo". The first real build on your machine is the remaining test; the code sticks to stable Spring APIs to keep that risk low.

### Exercises, in order

- [ ] **Build line 1, then line 2.** Run `mvn help:effective-pom` in `service-a` on both and compare the Java and Boot versions.
- [ ] **Read the migration diffs.** `git -C service-a diff on-platform-1 on-platform-2` and `git -C acme-test-support diff v1.0.0 v2.0.0`. Notice `acme-commons` changed no code.
- [ ] **Break the JDK rule.** Build line 2 with JDK 17 and read the enforcer message.
- [ ] **Simulate a CVE fix.** In `acme-parent`, add a Boot version property override, release `2.0.1` locally (`mvn versions:set`, `mvn install`), re-point `acme-service-parent`, bump `service-a`, and confirm with `mvn dependency:tree`.
- [ ] **Ship a library feature.** Add something to `acme-commons`, take it through 2.1.0 → `acme-bom` 2.1.0 → `service-b` following the shared-library scenario.
- [ ] **Try the trap.** Change `acme-service-parent` to import `acme-bom` with `${project.version}`, rebuild a service, and read the error.
- [ ] **Push to real remotes.** Create seven repos on GitHub/GitLab, push with tags, enable Renovate, and watch it open a platform PR.

## Further reading

**Start with the two official Maven guides, then the Spring Boot pages; together they cover every mechanism this guide relies on.**

| # | Read | What you'll get from it |
| --- | --- | --- |
| 1 | [Maven: Introduction to the POM](https://maven.apache.org/guides/introduction/introduction-to-the-pom.html) | Super POM, inheritance vs aggregation, interpolation (the root of the `${project.version}` trap) |
| 2 | [Maven: Introduction to the Dependency Mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html) | Transitive dependencies, scopes, `dependencyManagement`, importing BOMs |
| 3 | [Maven: POM Reference](https://maven.apache.org/pom.html) | Every POM element, for looking things up |
| 4 | [Spring Boot Maven Plugin: Using the Plugin](https://docs.spring.io/spring-boot/maven-plugin/using.html) | What `spring-boot-starter-parent` gives you, property overrides, using Boot without its parent |
| 5 | [Spring Boot: Creating Your Own Auto-configuration](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html) | How to build shared libraries like `acme-commons` as proper starters |
| 6 | [Spring Boot wiki: Maven POM Files](https://github.com/spring-projects/spring-boot/wiki/Maven-POM-Files) | How Spring Boot itself separated a public BOM from an internal parent |
| 7 | [Baeldung: Spring Boot Starter Parent](https://www.baeldung.com/spring-boot-starter-parent) | A gentle walk through the starter parent |
| 8 | [Baeldung: Spring Boot Without the Parent POM](https://www.baeldung.com/spring-boot-dependency-management-custom-parent) | Using Boot under a corporate parent |
| 9 | [Baeldung: Spring with Maven BOM](https://www.baeldung.com/spring-maven-bom) | BOM basics and why imports beat single inheritance |
| 10 | [Spring Boot 4.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide) | Everything that changes in the Boot 3 → 4 move |
| 11 | [Maven Enforcer: built-in rules](https://maven.apache.org/enforcer/enforcer-rules/index.html) | The guardrails used in `acme-parent` and more |
| 12 | [Renovate: noise reduction](https://docs.renovatebot.com/noise-reduction/) | Grouping and auto-merging platform bumps |
