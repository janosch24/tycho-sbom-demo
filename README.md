# tycho-sbom-demo

Minimal reproducible demo for a Tycho SBOM-plugin resolution issue:

> **Embedded OSGi `Bundle-ClassPath` dependencies only ever get a full Maven
> identity (group/artifactId/version + `pkg:maven` purl) in the generated SBOM
> if the exact same artifact can also be downloaded byte-identically from
> `https://repo.maven.apache.org/maven2/`. Correctly embedded Maven metadata
> (`pom.xml`/`pom.properties`) alone is not enough — the resolver discards it
> again if this final, hardcoded Central-only verification fails.**

This started out as a narrower report about `<parent>`-inherited `groupId`/
`version` in an embedded `pom.xml` (see git history / the original issue text
below). Systematic testing with three differently-sourced libraries showed
that hypothesis was incomplete — see [Root cause](#root-cause) below for the
corrected explanation, and [`docs/issue-body.md`](docs/issue-body.md) for the
full, up-to-date write-up.

Versions pinned for reproducibility:

| Component | Version | Hosted on | Embeds `pom.xml`/`pom.properties`? |
|---|---|---|---|
| Tycho | **5.0.3** | – | – |
| Eclipse release repo | 2024-06 | – | – |
| `com.diogonunes:JColor` | 5.2.0 | Maven Central | both, explicit GAV |
| `org.jzy3d:jzy3d-jdt-core` | 2.2.0 | `maven.jzy3d.org` (**not** Central) | both, GAV via `<parent>` |
| `com.hivemq:hivemq-mqtt-client` | 1.3.16 | Maven Central | neither (Gradle build) |

---

## Module architecture

```
tycho-sbom-demo/
├── pom.xml                              # parent – Tycho 5.0.3 + shared config
├── com.example.sbom.demo/               # OSGi eclipse-plugin bundle
│   ├── pom.xml                          # copies embedded JARs via maven-dependency-plugin
│   ├── build.properties                 # Tycho bin.includes – packs lib/ into bundle
│   └── META-INF/
│       └── MANIFEST.MF                  # Bundle-ClassPath with all embedded JARs
└── com.example.sbom.repository/        # eclipse-repository (p2 update-site + product)
    ├── pom.xml                          # tycho-sbom-plugin:generator configured HERE
    ├── category.xml                     # includes com.example.sbom.demo bundle
    └── com.example.sbom.demo.product    # minimal product → drives materialize-products
```

**Update site vs. installed product**

Tycho's `eclipse-repository` packaging builds two distinct artefacts:

| Artefact | Path | Layout |
|---|---|---|
| p2 update-site | `target/repository/` | `features/`, `plugins/`, `artifacts.jar`, `content.jar` |
| Installed product | `target/products/com.example.sbom.demo.product/linux/gtk/x86_64/` | `configuration/config.ini`, `plugins/`, … |

The `tycho-sbom-plugin:generator` goal requires an **installed product** layout,
not the plain update-site. It reads `configuration/config.ini` to discover which
bundles are present. If you point it at the update-site directory, it throws
`NoSuchFileException` because `configuration/config.ini` does not exist there.

**Where `configuration/config.ini` comes from**

The `.product` file (`com.example.sbom.demo.product`) tells Tycho to run
`tycho-p2-director-plugin:materialize-products` during the `package` phase.
The p2 director installs the listed plugins and writes `configuration/config.ini`
(with the `osgi.framework` and `osgi.bundles` properties) into the product
directory. This is the same file created when you install Eclipse normally.

`com.example.sbom.demo/lib/` is **generated at build time** by
`maven-dependency-plugin` and is not committed to version control.

---

## Prerequisites

| Tool | Required version |
|---|---|
| Java | **21+** (Tycho 5.0.3 SBOM plugin requires Java 21) |
| Maven | 3.9+ |
| Network | access to Maven Central, `download.eclipse.org`, and `maven.jzy3d.org` |

> **Note:** Tycho 5.0.3 and its `tycho-sbom-plugin` are compiled for Java 21
> (class-file version 65). The build will fail with
> `UnsupportedClassVersionError` on Java 17 or earlier.

---

## Build & reproduce the SBOM discrepancy

### Run the build

```bash
mvn -V -e -DskipTests clean verify
```

The `package` phase builds the p2 update-site **and** materializes the Eclipse
product (creating `configuration/config.ini`). The `verify` phase then runs the
SBOM generator against the installed product.

Optionally add `-Dcentral-search=true` (maps to `<centralsearch>true</centralsearch>`)
to also enable filename-/hash-based lookup against Maven Central — see
[What `centralsearch` actually changes](#what-centralsearch-actually-changes) below.

### Verify the installation exists and contains `config.ini`

After `mvn verify` completes, confirm the installation was created:

```bash
ls com.example.sbom.repository/target/products/com.example.sbom.demo.product/linux/gtk/x86_64/configuration/config.ini
grep osgi.bundles \
  com.example.sbom.repository/target/products/com.example.sbom.demo.product/linux/gtk/x86_64/configuration/config.ini
```

### Output locations

| Artifact | Path |
|---|---|
| OSGi bundle JAR | `com.example.sbom.demo/target/com.example.sbom.demo-1.0.0-SNAPSHOT.jar` |
| p2 update-site | `com.example.sbom.repository/target/repository/` |
| Installed product | `com.example.sbom.repository/target/products/com.example.sbom.demo.product/linux/gtk/x86_64/` |
| Generated SBOM | `com.example.sbom.repository/target/com.example.sbom.repository.json` *(CycloneDX JSON)* |

### Expected behaviour

All three embedded dependencies appear as components containing group, name,
version and a `pkg:maven` purl:

```json
{ "type": "library", "group": "com.diogonunes", "name": "JColor",             "version": "5.2.0",  "purl": "pkg:maven/com.diogonunes/JColor@5.2.0" }
{ "type": "library", "group": "org.jzy3d",       "name": "jzy3d-jdt-core",     "version": "2.2.0",  "purl": "pkg:maven/org.jzy3d/jzy3d-jdt-core@2.2.0" }
{ "type": "library", "group": "com.hivemq",       "name": "hivemq-mqtt-client", "version": "1.3.16", "purl": "pkg:maven/com.hivemq/hivemq-mqtt-client@1.3.16" }
```

### Actual behaviour (default configuration, `centralsearch` unset)

| Library | Resolved? |
|---|---|
| JColor | ✅ yes |
| jzy3d-jdt-core | ❌ no — `name` is the raw jar path, no group/version/purl |
| hivemq-mqtt-client | ❌ no — `version` is present (parsed from the filename) but no group/purl |

Example of an unresolved entry:

```json
{
  "type": "library",
  "bom-ref": "plugins/com.example.sbom.demo_....jar^lib/jzy3d-jdt-core-2.2.0.jar",
  "name": "lib/jzy3d-jdt-core-2.2.0.jar"
}
```

---

## Inspecting the embedded JAR metadata

Run `mvn verify` at least once first so `com.example.sbom.demo/lib/` is populated.

### JColor — resolved ✅

```bash
jar tf com.example.sbom.demo/lib/JColor-5.2.0.jar | grep META-INF/maven
unzip -p com.example.sbom.demo/lib/JColor-5.2.0.jar META-INF/maven/com.diogonunes/JColor/pom.properties
unzip -p com.example.sbom.demo/lib/JColor-5.2.0.jar META-INF/maven/com.diogonunes/JColor/pom.xml
```

`pom.xml` declares `<groupId>`/`<version>` explicitly. Central hosts this
artifact, so the final verification step succeeds.

### jzy3d-jdt-core — not resolved ❌

```bash
jar tf com.example.sbom.demo/lib/jzy3d-jdt-core-2.2.0.jar | grep META-INF/maven
unzip -p com.example.sbom.demo/lib/jzy3d-jdt-core-2.2.0.jar META-INF/maven/org.jzy3d/jzy3d-jdt-core/pom.properties
unzip -p com.example.sbom.demo/lib/jzy3d-jdt-core-2.2.0.jar META-INF/maven/org.jzy3d/jzy3d-jdt-core/pom.xml
```

`pom.xml` inherits `groupId`/`version` from a `<parent>` element — but
`pom.properties` (which the resolver actually reads for embedded jars, **not**
`pom.xml`, see [Root cause](#root-cause)) already contains the full GAV. The
real reason this stays unresolved is that this artifact is published on
`maven.jzy3d.org`, not on Maven Central.

### hivemq-mqtt-client — not (fully) resolved ❌

```bash
jar tf com.example.sbom.demo/lib/hivemq-mqtt-client-1.3.16.jar | grep META-INF/maven
```

This prints nothing — the jar contains **no** `META-INF/maven` directory at
all, because it was built and published with Gradle. Gradle's `maven-publish`
plugin does not embed `pom.xml`/`pom.properties` into the jar; that is purely
a convention of Maven's own archiver. Without `centralsearch=true`, the
resolver therefore has no candidate GAV to try at all for this artifact.

---

## Root cause

The originally suspected cause — that the resolver reads `pom.xml` but not
`pom.properties`, so `<parent>`-inherited `groupId`/`version` breaks resolution
— **does not match the actual code**. In
[`MavenDescriptor.java`](https://github.com/eclipse-cbi/p2repo-sbom/blob/main/plugins/org.eclipse.cbi.p2repo.sbom/src/org/eclipse/cbi/p2repo/sbom/MavenDescriptor.java),
`createFromBytes(...)` — the method used to resolve embedded Bundle-ClassPath
jars — reads **only** `META-INF/maven/.../pom.properties`; it never parses the
embedded `pom.xml` at all. Since `pom.properties` always contains the fully
resolved GAV regardless of `<parent>` inheritance, this step succeeds for both
JColor and jzy3d-jdt-core.

The actual point of failure is later, in `SBOMGenerator.setMavenPurl(...)`:

```java
var mavenArtifactBytes = contentHandler.getBinaryContent(mavenDescriptor.toArtifactURI());
...
if (equivalent(bytes, mavenArtifactBytes, differences)) {
    component.setPurl(mavenDescriptor.mavenPURL());
    return true;
}
```

`toArtifactURI()` is built from a hardcoded constant in `MavenDescriptor.java`:

```java
public static final String MAVEN_CENTRAL_URI = "https://repo.maven.apache.org/maven2/";
```

A correctly identified GAV (from `pom.properties`, from filename search, or
from hash search) is only kept — group/artifactId/version + `pkg:maven` purl
set — if the *exact same bytes* can also be downloaded from that fixed
Central URL. If that download 404s (because the artifact is hosted anywhere
else), the candidate is silently discarded and the component falls back to a
bare, path-named entry. This is why JColor (on Central) resolves, while
jzy3d-jdt-core (on `maven.jzy3d.org`) does not, even though both are correctly
identified from `pom.properties` in exactly the same way.

### What `centralsearch` actually changes

`centralsearch` (`-central-search`) only affects **how a candidate GAV is
found**, not the verification step above:

- `MavenDescriptor.createFromJarName(...)` — tried first for embedded jars,
  parses the jar's **filename** and queries `search.maven.org` by
  artifactId/version. Only runs when `centralsearch=true`; this is what gives
  `hivemq-mqtt-client` (no embedded metadata at all) a candidate GAV.
- The hash-based fallback inside `createFromBytes(...)` (queries
  `central.sonatype.com` by SHA-1) — same restriction.

Neither path changes `setMavenPurl`'s hardcoded Central-only verification.
That is why `jzy3d-jdt-core` stays unresolved with `centralsearch=true` as well
— it never had a metadata problem, it has a hosting-location problem.

---

## Proposed fix

`p2repo-sbom` already ships a redirection mechanism for exactly this class of
problem: `URIUtil.URIMap` / `parseRedirections(...)`, wired into
`ContentHandler.getBinaryContent(...)` (which `setMavenPurl` calls), activated
via the generator's own `-content-redirections "source->target"` CLI flag
(prefix-based, both sides must end in `/`).

`tycho-sbom-plugin`'s `GeneratorMojo` does not currently expose this flag. A
small, additive change fixes that — see
[`docs/issue-body.md`](docs/issue-body.md) for the concrete patch and a
validated example configuration that resolves both `jzy3d-jdt-core` (redirected
to `maven.jzy3d.org`) and an internal, non-public artifact (redirected to a
private Artifactory instance) without any change to `p2repo-sbom` itself.

---

## Troubleshooting

### `Could not find goal 'sbom-maven' in plugin org.eclipse.tycho:tycho-sbom`

**Cause:** Wrong artifactId and goal name. `tycho-sbom` (without `-plugin`) is not
the correct artifact for Tycho 5.x, and `sbom-maven` is not a valid goal.

**Fix:** Use `tycho-sbom-plugin` with goal `generator`.

---

### `Could not find goal 'cyclonedx' in plugin org.eclipse.tycho:tycho-sbom`

**Cause:** Same wrong artifactId; `cyclonedx` was never a goal in this plugin.

**Fix:** Use `tycho-sbom-plugin` with goal `generator`.

---

### `NoSuchFileException: …/configuration/config.ini`

**Cause:** The `installation` parameter in `tycho-sbom-plugin:generator` points
to the p2 update-site directory (`target/repository/`). That directory is an
update-site — it contains `features/`, `plugins/`, `artifacts.jar`, and
`content.jar`, but **no** `configuration/` directory. The generator cannot find
`configuration/config.ini`, which it requires to discover installed bundles.

**Fix:** Point `<installation>` to a **materialized product directory** produced
by `tycho-p2-director-plugin:materialize-products`, not to the update-site:

```xml
<!-- ✗ Wrong – update-site, no configuration/config.ini -->
<installation>${project.build.directory}/repository</installation>

<!-- ✓ Correct – installed product, includes configuration/config.ini -->
<installation>${project.build.directory}/products/com.example.sbom.demo.product/linux/gtk/x86_64</installation>
```

To have the director create the installation, add a `.product` file to the
`eclipse-repository` module. Tycho's `package` lifecycle then runs
`tycho-p2-publisher-plugin:publish-products` followed by
`tycho-p2-director-plugin:materialize-products` automatically.

---

### `One of 'installations' or 'installation' must be specified`

**Cause:** The `generator` goal was run on a `pom`-packaged parent or on an
`eclipse-plugin` module that has no built p2 installation to analyze.

**Fix:** Run `generator` only in an `eclipse-repository` module, and set the
`<installation>` parameter to the materialized product path (see above).

---

### `UnsupportedClassVersionError` for `GeneratorMojo` (class file version 65.0)

**Cause:** `tycho-sbom-plugin` 5.0.3 requires Java 21. You are running Java 17
or earlier.

**Fix:** Switch to Java 21+ (e.g. via `JAVA_HOME` or `jenv`).

---

### An artifact stays unresolved even though `pom.properties` is complete and correct

**Cause:** This is the actual bug described above — the artifact is not hosted
on `https://repo.maven.apache.org/maven2/`, so `setMavenPurl`'s mandatory
verification download fails and the correctly identified GAV is discarded.
`centralsearch` does not help here, since it only affects candidate discovery,
not verification.

**Fix / workaround:** Configure `-content-redirections` (once your
`tycho-sbom-plugin` version supports it, see [Proposed fix](#proposed-fix)) to
point the verification lookup at the repository the artifact is actually
hosted on.

---

### Build fails resolving p2 dependencies

**Cause:** Network issues reaching `download.eclipse.org`.

**Fix:** Ensure outbound HTTPS access to `download.eclipse.org` and Maven Central.
You can also mirror the Eclipse 2024-06 p2 repository locally.
