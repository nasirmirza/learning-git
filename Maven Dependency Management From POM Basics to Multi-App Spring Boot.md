# Maven Dependency Management: From POM Basics to Multi-App Spring Boot

Oct 6, 2026 · @Nas Mir

## How to use this guide

The short version of everything below: **declare versions in one place, inherit or import them everywhere else, and let a build rule fail when someone drifts.** Parts 1–3 teach how Maven decides which jar ends up on your classpath. Parts 4–6 teach the two sharing mechanisms (parent POMs and BOMs) and how Spring Boot uses them. Parts 7–9 apply all of it to a fleet of Spring Boot apps that share in-house libraries.

Suggested path: read Parts 1–3 with a small project open and run `mvn dependency:tree` after every change. Then build the two-repo setup in Part 7 yourself; it is the fastest way to make the parent-vs-BOM distinction stick.

Assumes Maven 3.9.x (Maven 4 is noted where it differs) and Spring Boot 4.x, which needs Java 17 or later.

## Part 1 — POM fundamentals

A POM (Project Object Model) is the `pom.xml` that tells Maven what your project is, what it needs, and how to build it. Everything Maven does starts by computing the **effective POM**: your file merged with its parents and Maven's built-in super POM.

### 1.1 Coordinates (GAV)

Every artifact is identified by three coordinates, plus optional packaging and classifier.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.acme.orders</groupId>   <!-- organisation / namespace -->
  <artifactId>order-service</artifactId> <!-- module name -->
  <version>1.4.0-SNAPSHOT</version>     <!-- SNAPSHOT = mutable dev build -->
  <packaging>jar</packaging>            <!-- jar (default), war, pom -->
</project>
```

| Element | Meaning | Tip |
| --- | --- | --- |
| groupId | Reverse-domain namespace | One per team or product, e.g. `com.acme.platform` |
| artifactId | Name of this module | Becomes the jar file name |
| version | Release or `-SNAPSHOT` | Releases are immutable once deployed; snapshots are not |
| packaging | Output type | `pom` for parents, BOMs and aggregators |
| classifier | Variant of the same GAV | e.g. `sources`, `tests`, `linux-x86_64` |

### 1.2 Declaring a dependency

```xml
<dependencies>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.17.2</version>
  </dependency>
</dependencies>
```

Maven downloads it from a repository (Maven Central by default, or your company's Nexus/Artifactory) into `~/.m2/repository`, then puts it on the right classpaths according to its **scope**.

### 1.3 Scopes

Scope decides which classpath a jar lands on and whether it travels to consumers.

| Scope | Compile | Test | Runtime / packaged | Passed to consumers | Typical use |
| --- | --- | --- | --- | --- | --- |
| compile (default) | yes | yes | yes | yes, as compile | Your normal libraries |
| provided | yes | yes | no | no | Servlet API in a WAR, Lombok |
| runtime | no | yes | yes | yes, as runtime | JDBC drivers |
| test | no | yes | no | no | JUnit, Mockito, Testcontainers |
| system | yes | yes | no | no | Avoid; points at a local file path |
| import | — | — | — | — | Only inside `dependencyManagement`, for BOMs (Part 5) |

Rule of thumb for shared libraries: anything the library only needs to compile against, and that every consuming app already has (Spring itself, for instance), is a good candidate for `provided` or `optional`, so the library does not force a version onto apps.

### 1.4 Repositories and the lifecycle in one paragraph

`mvn install` builds the jar and copies it into your local `~/.m2`, so other projects on your machine can use it. `mvn deploy` uploads it to a remote repository (`distributionManagement`), so other teams and CI can use it. Shared libraries live or die on this: an app can only depend on a library version that has been deployed somewhere it can reach.

## Part 2 — Transitive dependencies and conflict resolution

When two paths bring in different versions of the same library, Maven keeps exactly one: **the nearest declaration wins, and on a tie the first one declared wins.** It does not pick the highest version. This single rule explains most "it works on my module but not in the app" bugs.

### 2.1 Transitive dependencies

If your app depends on `lib-a`, and `lib-a` depends on `guava`, then `guava` is on your classpath too. Maven walks the whole graph using each dependency's own POM.

What travels transitively, by scope:

- `compile` deps of a dependency arrive as `compile`.
- `runtime` deps arrive as `runtime`.
- `provided`, `test` and `optional` deps do **not** arrive at all.

### 2.2 Dependency mediation: nearest wins

```text
my-app
├── lib-a:1.0
│   └── guava:31.0        (depth 2)
└── lib-b:1.0
    └── lib-c:1.0
        └── guava:33.0    (depth 3)
```

Maven picks `guava:31.0` because it is closer to the root, even though 33.0 is newer. If `lib-c` calls a method added in 32.x you get a `NoSuchMethodError` at runtime, not at compile time.

Two ways to take control:

1. Declare the version yourself (depth 1 always wins) — but this adds a direct dependency you may not use.
2. Better: pin it in `dependencyManagement` (Part 3), which sets the version wherever guava appears in the graph, at any depth, without adding it as a direct dependency.

### 2.3 Exclusions

Cut a transitive branch you don't want — for example, swapping Tomcat for Jetty, or removing an old logging bridge.

```xml
<dependency>
  <groupId>com.acme.platform</groupId>
  <artifactId>acme-http-client</artifactId>
  <exclusions>
    <exclusion>
      <groupId>commons-logging</groupId>
      <artifactId>commons-logging</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

Wildcards (`<groupId>*</groupId>`) are allowed but hide what you removed. Prefer naming each exclusion, and add a comment saying why.

### 2.4 Optional dependencies

`<optional>true</optional>` means "I compile against this, but consumers don't get it unless they declare it themselves." It is how a shared library supports several backends without forcing all of them on every app.

```xml
<!-- in acme-cache-lib: supports Redis if the app brings it -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
  <optional>true</optional>
</dependency>
```

This pairs naturally with Spring Boot's `@ConditionalOnClass`: the auto-configuration only activates when the app has actually added Redis.

### 2.5 Seeing what really happened

```bash
mvn dependency:tree                       # resolved graph
mvn dependency:tree -Dverbose             # also shows omitted versions and why
mvn dependency:tree -Dincludes=com.google.guava:guava
mvn help:effective-pom                    # the merged POM Maven actually uses
```

In verbose output, look for `omitted for conflict with 33.0` — that line tells you which version lost and where it came from.

## Part 3 — Properties, dependencyManagement and pluginManagement

`dependencyManagement` is the most important element in this guide: **it declares versions (and scopes, exclusions) without adding any dependency.** A dependency only appears when a module lists it in `<dependencies>`, and then it can omit the version.

### 3.1 Properties

```xml
<properties>
  <java.version>21</java.version>
  <maven.compiler.release>${java.version}</maven.compiler.release>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <guava.version>33.3.1-jre</guava.version>
</properties>
```

Properties are inherited by child POMs and can be overridden there. That makes them the lever for "change one number, update everywhere" — and, as Part 6 shows, the way to override a Spring Boot-managed version.

### 3.2 dependencyManagement vs dependencies

```xml
<!-- parent or BOM -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.google.guava</groupId>
      <artifactId>guava</artifactId>
      <version>${guava.version}</version>
    </dependency>
  </dependencies>
</dependencyManagement>

<!-- child module: no version needed -->
<dependencies>
  <dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
  </dependency>
</dependencies>
```

|  | `<dependencies>` | `<dependencyManagement>` |
| --- | --- | --- |
| Adds the jar to the build | Yes | No |
| Inherited by children | Yes — every child gets it | Yes — as version rules only |
| Affects transitive versions | Only via nearest-wins | Yes — overrides the version at any depth |
| Typical place | Each app/module | Parent POM or BOM |

The third row is the key insight: a managed version beats mediation. If your parent manages `guava:33.3.1-jre`, every module gets that version even when guava only arrives transitively four levels deep.

### 3.3 pluginManagement vs plugins

The same split exists for build plugins. `pluginManagement` fixes plugin versions and default configuration; a module activates a plugin by listing it under `<build><plugins>`.

```xml
<build>
  <pluginManagement>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>3.5.2</version>
      </plugin>
    </plugins>
  </pluginManagement>
</build>
```

Always pin plugin versions. Unpinned plugins fall back to whatever the Maven version defaults to, so two developers on different Maven versions get different builds.

### 3.4 Version ranges — and why to avoid them

Maven accepts ranges such as `[1.2,2.0)`, but they make builds non-reproducible: the same commit can resolve differently next week. Use exact versions everywhere, and let automation (Part 8) propose upgrades.

## Part 4 — Parent POMs: inheritance vs aggregation

A parent POM is a `pom`-packaged project whose configuration children inherit. **Inheritance (`<parent>`) and aggregation (`<modules>`) are separate features** that are often, but not necessarily, used together.

### 4.1 What a child inherits

Inherited: `groupId`, `version`, `properties`, `dependencies`, `dependencyManagement`, `build` (plugins and `pluginManagement`), `repositories`, `distributionManagement`, `profiles`, `reporting`.

Not inherited: `artifactId`, `name`, `packaging`, `modules`, `prerequisites`.

```xml
<parent>
  <groupId>com.acme</groupId>
  <artifactId>acme-parent</artifactId>
  <version>5.2.0</version>
  <relativePath/>  <!-- empty: always resolve from a repository -->
</parent>
<artifactId>order-service</artifactId>
```

**relativePath**: by default Maven first looks for the parent at `../pom.xml`. In a multi-module repo that is what you want. When the parent lives in a different repo (a corporate parent), set `<relativePath/>` empty so Maven never picks up some unrelated `pom.xml` from the folder above.

### 4.2 Aggregation: building many modules together

An aggregator lists modules so one `mvn verify` builds them all in dependency order (the "reactor").

```xml
<!-- order-system/pom.xml -->
<packaging>pom</packaging>
<modules>
  <module>order-api</module>
  <module>order-domain</module>
  <module>order-service</module>
</modules>
```

|  | Inheritance (`<parent>`) | Aggregation (`<modules>`) |
| --- | --- | --- |
| Direction | Child points to parent | Aggregator points to modules |
| Purpose | Share configuration | Build several modules in one command |
| Can cross repos | Yes (via a repository) | No — modules are folders |
| Typical in a monorepo | Both, in the same root POM | Both, in the same root POM |

Useful reactor flags: `-pl order-service -am` builds one module plus what it needs; `-rf :order-service` resumes after a failure.

### 4.3 Single parent only

Maven allows exactly one `<parent>`. This is the main limitation of parents and the reason BOMs exist: if your app must inherit from the corporate parent, it cannot *also* inherit from `spring-boot-starter-parent` — unless the corporate parent itself extends it (Part 7).

### 4.4 Versioning a multi-module project

Repeating the version in every child is error-prone. Two clean options:

- **CI-friendly versions**: use `${revision}` in the root and children, set it with `-Drevision=1.4.0`, and add the `flatten-maven-plugin` so the deployed POMs contain real versions, not a placeholder.
- **versions-maven-plugin**: `mvn versions:set -DnewVersion=1.4.0` rewrites every module in one go.

Maven 4 removes most of this pain: children can omit the parent version, and the consumer POM it deploys is flattened for you. Maven 3.9 remains the safe default for most teams today, so plan for the flatten plugin until you migrate.

### 4.5 Debugging inheritance

`mvn help:effective-pom -Dverbose` prints the merged POM with comments showing which file each line came from. Use it whenever a version or plugin setting surprises you.

## Part 5 — BOMs and import scope

A BOM (Bill of Materials) is a `pom`-packaged artifact that contains **only** a `dependencyManagement` section. Importing it copies those version rules into your project — and unlike a parent, **you can import as many BOMs as you like.**

### 5.1 Writing a BOM

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.acme.platform</groupId>
  <artifactId>acme-platform-bom</artifactId>
  <version>2026.10.0</version>
  <packaging>pom</packaging>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>com.acme.platform</groupId>
        <artifactId>acme-security-lib</artifactId>
        <version>3.1.0</version>
      </dependency>
      <dependency>
        <groupId>com.acme.platform</groupId>
        <artifactId>acme-observability-lib</artifactId>
        <version>2.4.2</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
</project>
```

### 5.2 Importing a BOM

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.acme.platform</groupId>
      <artifactId>acme-platform-bom</artifactId>
      <version>2026.10.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

`type=pom` + `scope=import` is the only valid combination, and it only works inside `dependencyManagement`.

### 5.3 Precedence rules (memorise these)

1. A version written directly in your own `dependencyManagement` beats anything imported.
2. Between imported BOMs, **the first one declared wins** for a given artifact. Order matters: put the BOM whose versions you trust most first.
3. Properties inside an imported BOM are resolved inside the BOM. Setting `<jackson.version>` in your project does **not** change a version that came from an imported BOM — you must re-declare the artifact (or use a parent, where property overrides do work).
4. A version in a module's `<dependencies>` overrides managed versions for that module only. Avoid this in apps; it is how drift starts.

### 5.4 Parent vs BOM

| Need | Parent POM | BOM |
| --- | --- | --- |
| Share dependency versions | Yes | Yes |
| Share plugins, plugin config, compiler settings | Yes | No |
| Share properties, profiles, repositories | Yes | No |
| Combine several | No — one parent | Yes — import many |
| Override a version via a property | Yes | No (re-declare instead) |
| Safe for external consumers of a library | Risky — forces your build on them | Yes |

The common pattern is to use both: a **parent** for build conventions, and one or more **BOMs** for versions. Libraries that other teams consume should publish a BOM, not require a parent.

## Part 6 — Spring Boot's dependency management

Spring Boot ships one BOM, `spring-boot-dependencies`, that pins several hundred libraries (Spring Framework, Jackson, Hibernate, Tomcat, drivers, test libraries) to versions tested together. **Choosing a Spring Boot version is really choosing that whole set.** You consume it in one of two ways.

### 6.1 Option A — inherit `spring-boot-starter-parent`

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>4.1.1</version>
  <relativePath/>
</parent>
```

The starter parent itself extends `spring-boot-dependencies`, then adds build conventions: Java release level from `java.version`, UTF-8 encoding, the `-parameters` compiler flag, resource filtering with `@..@` placeholders, and pre-configured `spring-boot-maven-plugin` executions (repackage) plus versions for common plugins.

Overriding a managed version is one property:

```xml
<properties>
  <java.version>21</java.version>
  <postgresql.version>42.7.5</postgresql.version>  <!-- overrides Boot's choice -->
</properties>
```

The property names are listed in the Spring Boot reference under "Dependency Versions — Version Properties". Override sparingly and only for patches, typically security fixes.

### 6.2 Option B — import the BOM (no parent)

Use this when you already need a different parent, such as a corporate one.

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>${spring-boot.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<build>
  <plugins>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
      <version>${spring-boot.version}</version>
      <executions>
        <execution>
          <goals><goal>repackage</goal></goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

What you lose versus the parent: the plugin defaults above (you must configure them yourself), and **property-based overrides**. To override a version you now declare the artifact in your own `dependencyManagement` *before* the Boot BOM import, because your own entries win (Part 5.3).

```xml
<dependencyManagement>
  <dependencies>
    <!-- override first -->
    <dependency>
      <groupId>org.postgresql</groupId>
      <artifactId>postgresql</artifactId>
      <version>42.7.5</version>
    </dependency>
    <!-- then the Boot BOM import -->
  </dependencies>
</dependencyManagement>
```

Some libraries ship their own BOM (Jackson, Netty, Testcontainers). For those, import the newer library BOM *above* the Spring Boot BOM so it wins for every module in that family.

### 6.3 Other Spring BOMs that must match Boot

Spring Cloud, Spring AI and Spring Modulith publish their own BOMs, and each release only supports certain Boot versions. Spring Cloud uses calendar release trains (e.g. `2025.1.x`); check the compatibility table on the Spring Cloud project page before every Boot upgrade, and bump the train in the same change.

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-dependencies</artifactId>
  <version>${spring-cloud.version}</version>
  <type>pom</type>
  <scope>import</scope>
</dependency>
```

### 6.4 Where the Boot version stands (Oct 2026)

Per [spring.io](https://spring.io/projects/spring-boot), the current GA line is 4.1.x. The 4.0 line arrived in November 2025 on Spring Framework 7, and the 3.5 line's open-source support ended on 30 June 2026 ([HeroDevs summary](https://www.herodevs.com/blog-posts/spring-boot-versions-eol-dates-and-latest-releases-april-2026)). Boot ships a new minor roughly every May and November, and each minor gets about 12 months of free support — so a fleet of apps should plan on **two Boot minor upgrades a year**.

## Part 7 — Many Spring Boot apps sharing libraries

The setup that scales: **one platform repo publishes a platform BOM and a corporate parent; every app inherits that parent and declares no versions of its own.** Upgrading Spring Boot or a shared library then means bumping one parent version in each app.

### 7.1 The building blocks

| Artifact | Packaging | Contains | Who uses it |
| --- | --- | --- | --- |
| `acme-platform-bom` | pom | Versions of every in-house library; imports the Spring Boot and Spring Cloud BOMs | Apps (via parent) and any project that cannot use the parent |
| `acme-parent` | pom | Extends `spring-boot-starter-parent`; imports `acme-platform-bom`; plugin config, enforcer rules, repositories | Every app |
| Shared libraries, e.g. `acme-security-lib` | jar | Reusable code, ideally packaged as Spring Boot starters with auto-configuration | Apps |
| Apps, e.g. `order-service` | jar | Business code; `<parent>` = `acme-parent` | — |

&#91;embedded content: platform layout · solid = inherits, dashed = imports\]

Solid arrows are parent inheritance, dashed arrows are BOM imports; apps touch only `acme-parent`, and every version reaches them through it.

### 7.2 The platform BOM

```xml
<artifactId>acme-platform-bom</artifactId>
<version>2026.10.0</version>
<packaging>pom</packaging>

<properties>
  <spring-boot.version>4.1.1</spring-boot.version>
  <spring-cloud.version>2025.1.1</spring-cloud.version>
</properties>

<dependencyManagement>
  <dependencies>
    <!-- in-house libraries first, so they win -->
    <dependency>
      <groupId>com.acme.platform</groupId>
      <artifactId>acme-security-lib</artifactId>
      <version>4.3.0</version>
    </dependency>
    <!-- third-party pins that differ from Boot (security patches) -->
    <!-- then the framework BOMs -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>${spring-boot.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring-cloud.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

The version numbers here are illustrative; take the Spring Cloud train from its compatibility table for your Boot version.

### 7.3 The corporate parent

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>4.1.1</version>   <!-- must equal spring-boot.version in the BOM -->
  <relativePath/>
</parent>
<groupId>com.acme</groupId>
<artifactId>acme-parent</artifactId>
<version>2026.10.0</version>
<packaging>pom</packaging>

<properties>
  <java.version>21</java.version>
</properties>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.acme.platform</groupId>
      <artifactId>acme-platform-bom</artifactId>
      <version>${project.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
<!-- + pluginManagement, enforcer rules (Part 9), distributionManagement -->
```

Why extend the starter parent *and* import the BOM? The starter parent gives Boot's plugin conventions and property overrides; the platform BOM gives your libraries' versions. Keep the parent's Boot version and the BOM's `spring-boot.version` identical — the platform repo's own CI should fail if they differ.

**Gotcha — where third-party overrides go.** Maven merges inherited `dependencyManagement` before it processes imports, and an import only fills in artifacts not already managed. Because `acme-parent` already inherits Boot's versions from the starter parent, a third-party pin inside `acme-platform-bom` (say, a patched PostgreSQL driver) is silently ignored for apps on the parent. Put those overrides in `acme-parent` as Boot version properties (`<postgresql.version>`); keep the pins in the BOM only for projects that import the BOM without the parent. Verify with `mvn help:effective-pom`.

### 7.4 An app's POM

```xml
<parent>
  <groupId>com.acme</groupId>
  <artifactId>acme-parent</artifactId>
  <version>2026.10.0</version>
  <relativePath/>
</parent>
<artifactId>order-service</artifactId>

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webmvc</artifactId>
  </dependency>
  <dependency>
    <groupId>com.acme.platform</groupId>
    <artifactId>acme-security-lib</artifactId>
  </dependency>
</dependencies>
```

No versions anywhere. The parent version is the app's single dial for Spring Boot, Spring Cloud and every shared library.

### 7.5 Rules for the shared libraries

1. **Build them against the same Boot line the platform ships.** The library modules in the platform repo use a library parent that imports the same Boot BOM, so they compile against exactly what apps run.
2. **Don't pin Spring-managed versions inside a library.** Leave Jackson, Spring and friends version-less; the app's managed versions decide at runtime anyway (Part 3.2).
3. **Use `optional` for integrations** (Redis, Kafka) and Spring Boot `@ConditionalOnClass` auto-configuration, so apps only get what they use.
4. **Match library major versions to Boot majors.** Code compiled for Boot 4 usually fails on Boot 3 and vice versa. Keep a maintenance branch, e.g. `acme-security-lib` 3.x for Boot 3 and 4.x for Boot 4, during a migration.
5. **Release libraries from one place.** Simplest: one platform repo, libraries and BOM released together under the same calendar version. Independent library versions work too, but the BOM then becomes the only record of which combination was tested.

### 7.6 Monorepo or many repos?

| Layout | Strength | Cost |
| --- | --- | --- |
| All apps + libs in one repo, one root POM | Atomic upgrades; one build proves everything compiles | Large builds; all teams release together unless you add tooling |
| Platform repo + one repo per app (above) | Teams release independently; clear ownership | Apps lag behind the platform; needs automation (Part 8) |
| Every library its own repo | Maximum independence | Version matrix grows fast; avoid unless libraries are truly independent |

The platform-repo-plus-app-repos layout is the usual choice for several Spring Boot microservices.

## Part 8 — Keeping Spring Boot aligned across apps

Alignment is a process, not a file: **the platform team upgrades once, a bot opens the bump in every app, and a support policy limits how far any app may lag.** With the Part 7 layout, each app's upgrade is a one-line change to its parent version.

### 8.1 Version the platform as a release train

Give `acme-parent` and `acme-platform-bom` a calendar version such as `2026.10.0`, and publish a short mapping in the platform README:

| Platform version | Spring Boot | Spring Cloud train | Java | Status |
| --- | --- | --- | --- | --- |
| 2026.10.x | 4.1.x | per compatibility table | 21 | Current |
| 2026.04.x | 4.0.x | per compatibility table | 21 | Supported until Boot 4.0 OSS end |
| 2025.x | 3.5.x | per compatibility table | 17 | Unsupported — migrate |

Patch releases (`2026.10.1`, `.2`) carry Boot patch upgrades and CVE fixes; they should be safe to take without code changes.

### 8.2 The upgrade workflow

1. **Track milestones early.** Build the platform repo against the next Boot release candidate on a branch, so library breakages surface before GA.
2. **Upgrade the platform.** Bump the starter parent and `spring-boot.version` together, update the Spring Cloud train, fix the shared libraries, release `acme-parent` + BOM.
3. **Prove it on canary apps.** Pick two or three representative apps (web, messaging, batch) and upgrade them first.
4. **Roll out to all apps.** A bot (Renovate or Dependabot) opens a PR in every app repo bumping the `acme-parent` version. Each app's CI decides whether it merges cleanly.
5. **Use automated refactoring for majors.** OpenRewrite has Spring Boot migration recipes (for example, 3.x to 4.x) that rewrite POMs, properties and code; run them through the Maven plugin rather than by hand.
6. **Enforce the lag limit.** E.g. "apps must be on a supported platform version", checked in CI or by a dashboard of parent versions across repos.

### 8.3 Automation snippets

Renovate, grouping all platform artifacts into one PR per app:

```json
{
  "packageRules": [
    {
      "matchPackagePrefixes": ["com.acme"],
      "groupName": "acme platform",
      "automerge": true,
      "matchUpdateTypes": ["patch"]
    }
  ]
}
```

Useful commands while upgrading:

```bash
mvn versions:display-parent-updates
mvn versions:display-dependency-updates
mvn versions:update-parent -DparentVersion=2026.10.0
mvn -U org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:RELEASE \
  -Drewrite.activeRecipes=<the Spring Boot migration recipe for your target version>
```

Look up the exact recipe name in the OpenRewrite docs for your target Boot version; names change between releases.

### 8.4 Handling the exceptions

- **An app needs a newer library than Boot ships** (a CVE fix): add a version property override in that app's POM with a comment and a ticket; remove it on the next platform release. Better still, ship the override in a platform patch release so every app gets it.
- **An app can't upgrade yet:** it stays on the previous platform line, which you keep patching until that Boot line's support ends. Don't let it drift onto a mix of old parent and new libraries.
- **A library needs a feature only in the newer Boot:** release it in the new library major line only (Part 7.5, rule 4).

## Part 9 — Enforcement and troubleshooting

Conventions erode unless the build checks them. **Put the Maven Enforcer Plugin in the corporate parent so every app fails fast on drift.**

### 9.1 Enforcer rules worth having

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-enforcer-plugin</artifactId>
  <executions>
    <execution>
      <id>enforce</id>
      <goals><goal>enforce</goal></goals>
      <configuration>
        <rules>
          <requireMavenVersion><version>[3.9.6,)</version></requireMavenVersion>
          <requireJavaVersion><version>[21,)</version></requireJavaVersion>
          <dependencyConvergence/>
          <banDuplicatePomDependencyVersions/>
          <banDynamicVersions/>
          <bannedDependencies>
            <excludes>
              <exclude>commons-logging:commons-logging</exclude>
              <exclude>log4j:log4j</exclude>
            </excludes>
          </bannedDependencies>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

| Rule | Catches |
| --- | --- |
| `dependencyConvergence` | Two paths asking for different versions of the same artifact that nothing manages — the silent nearest-wins case |
| `requireUpperBoundDeps` | A resolved version lower than one some dependency asked for (stricter alternative) |
| `banDynamicVersions` | Ranges, `LATEST`, `RELEASE`, and optionally SNAPSHOTs in releases |
| `bannedDependencies` | Known-bad or legacy artifacts |
| `banDuplicateClasses` (extra-enforcer-rules) | The same class shipped in two jars |

`dependencyConvergence` can be noisy at first; fix by adding the conflicting artifact to the platform BOM, which is exactly where it belongs.

### 9.2 Symptoms and causes

| Symptom | Usual cause | First command |
| --- | --- | --- |
| `NoSuchMethodError`, `NoClassDefFoundError` at runtime | Nearest-wins picked an older version | `mvn dependency:tree -Dverbose -Dincludes=<group>` |
| App works locally, fails in CI | Stale SNAPSHOT or different Maven version | `mvn -U verify`; check enforcer Maven version |
| Override property has no effect | Version came from an imported BOM, or the parent already manages it | `mvn help:effective-pom -Dverbose` |
| Two logging frameworks fight at startup | Transitive bridge (commons-logging, log4j) | Exclude it; ban it in enforcer |
| Spring Cloud fails on startup after a Boot bump | Train not compatible with the new Boot | Check the compatibility table; bump the train |
| Unused jars bloating the app | Declared but never used | `mvn dependency:analyze` |

### 9.3 Supply-chain hygiene

- Generate an SBOM per app with the CycloneDX Maven plugin and feed it to your vulnerability scanner.
- Mirror all repositories through Nexus or Artifactory (`settings.xml` `<mirror>`), so builds don't depend on public uptime and you can block known-bad versions centrally.
- Never deploy over an existing release version; treat releases as immutable.

## Cheat sheet and common pitfalls

### Commands

| Goal | Command |
| --- | --- |
| See the resolved graph | `mvn dependency:tree` |
| See conflicts and losers | `mvn dependency:tree -Dverbose` |
| Find one artifact | `mvn dependency:tree -Dincludes=g:a` |
| See merged config | `mvn help:effective-pom -Dverbose` |
| Unused / undeclared deps | `mvn dependency:analyze` |
| Available upgrades | `mvn versions:display-dependency-updates` |
| Bump parent | `mvn versions:update-parent` |
| Build one module + its deps | `mvn -pl <module> -am verify` |
| Force snapshot refresh | `mvn -U verify` |

### Pitfalls

1. **Assuming the newest version wins.** Maven picks the nearest. Manage the version instead.
2. **Versions in app POMs.** Every hard-coded version is a future drift. Apps should have none.
3. **Overriding a property for a version that came from an imported BOM.** It does nothing; re-declare the artifact or use the parent.
4. **Pinning third-party versions in a BOM that sits under a Boot parent.** Inherited management wins; use properties in the parent (Part 7.3).
5. **Wrong BOM order.** The first imported BOM wins for shared artifacts.
6. **Parent's Boot version ≠ BOM's `spring-boot.version`.** Two Boot versions in one build; check it in CI.
7. **Spring Cloud train not matching Boot.** Always bump them together.
8. **Libraries forcing versions on apps.** Keep library POMs version-less for Spring-managed artifacts; use `optional`.
9. **Unpinned plugins.** Builds change when Maven changes.
10. **Empty `relativePath` forgotten** for a corporate parent; Maven may pick up a stray `../pom.xml`.

### The one-paragraph summary

Scopes decide where a jar goes; nearest-wins decides which version; `dependencyManagement` overrides that choice. Parents share build configuration (one only); BOMs share versions (many). Spring Boot is a BOM plus a parent. For a fleet of apps, publish a platform BOM and a corporate parent that extends Boot's parent, keep apps version-less, and let a bot roll out each platform release.
