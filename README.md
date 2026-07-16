# tycho-sbom-demo

Minimal reproducible demo for Eclipse Tycho SBOM-plugin resolution issues.

> **Embedded OSGi `Bundle-ClassPath` dependencies are frequently missing their
> Maven identity (`group`, `purl`) in the generated SBOM — not because their
> embedded metadata is bad, but because `tycho-sbom-plugin` / `p2repo-sbom`
> can only accept a resolved GAV if the exact same artifact can additionally
> be downloaded and byte-verified against a hardcoded Maven Central URL (or,
> in one specific case, against a URL built from the wrong file extension).**

This supersedes the project's original hypothesis (that a `<parent>`-inherited
`groupId`/`version` in the embedded `pom.xml` was the cause). That hypothesis
turned out to be **wrong** — see [Root cause](#root-cause) below. The actual
resolver never even parses `pom.xml` for embedded jars; it only reads
`pom.properties`, and that file is complete and correct in every case tested
here.

Versions pinned for reproducibility:

| Component | Version |
|---|---|
| Tycho | **5.0.3** |
| Eclipse release repo | 2024-06 |
| `com.diogonunes:JColor` | 5.2.0 *(resolved correctly)* |
| `org.jzy3d:jzy3d-jdt-core` | 2.2.0 *(not resolved — third-party repo)* |
| `com.hivemq:hivemq-mqtt-client` | 1.3.16 *(not resolved by default — Gradle build, no embedded Maven metadata)* |
| `com.google.guava:failureaccess` | 1.0.1 *(not resolved — wrong file extension, see Root cause #4)* |

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

`lib/` is **generated at build time** by `maven-dependency-plugin` and is
not committed to version control.

---

## Prerequisites

| Tool | Required version |
|---|---|
| Java | **21+** (Tycho 5.0.3 SBOM plugin requires Java 21) |
| Maven | 3.9+ |
| Network | access to Maven Central + `download.eclipse.org` |

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

### Verify the installation exists and contains `config.ini`

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
| `configuration/config.ini` | `…/linux/gtk/x86_64/configuration/config.ini` *(created by director)* |
| Generated SBOM | `com.example.sbom.repository/target/com.example.sbom.repository.json` *(CycloneDX JSON)* |

### Expected behaviour

All embedded dependencies appear as components containing group, name, version
and purl in the SBOM, e.g.:

```json
{ "type": "library", "group": "com.diogonunes", "name": "JColor", "version": "5.2.0", "purl": "pkg:maven/com.diogonunes/JColor@5.2.0" }
```

### Actual behaviour (default configuration)

`jzy3d-jdt-core`, `hivemq-mqtt-client` and `failureaccess` all appear in the SBOM,
but **not** as resolved Maven components — `group` and `purl` are missing, and
`name` falls back to the internal JAR path instead of the artifactId:

```json
{
  "type": "library",
  "bom-ref": "plugins/com.example.sbom.demo_1.0.0....jar^lib/jzy3d-jdt-core-2.2.0.jar",
  "name": "lib/jzy3d-jdt-core-2.2.0.jar"
}
```

---

## Result matrix

| Library | `pom.xml` embedded | `pom.xml` GAV | `pom.properties` embedded | `pom.properties` GAV correct | Hosted on | Resolved? |
|---|---|---|---|---|---|---|
| JColor 5.2.0 | yes | flat | yes | yes | Maven Central | ✅ yes |
| jzy3d-jdt-core 2.2.0 | yes | inherited from `<parent>` | yes | yes | `maven.jzy3d.org` (third-party) | ❌ no |
| hivemq-mqtt-client 1.3.16 | no | n/a | no | n/a | Maven Central | ❌ no *(✅ with `centralsearch=true`)* |
| failureaccess 1.0.1 | no | n/a | yes | yes | Maven Central | ❌ no *(even with a correct `content-redirection` and byte-identical mirror copy)* |

The result correlates with **where the artifact is hosted and how its
candidate GAV was found** — not with whether `pom.xml`/`pom.properties` are
present or correct. JColor and jzy3d-jdt-core differ exactly in whether
`pom.xml` declares `groupId`/`version` directly or inherits them from a
`<parent>` — yet both are identified correctly from `pom.properties`. The
original `<parent>`-inheritance hypothesis from issue #6155 does not hold.

---

## Root cause

### #1 — the original `<parent>`-inheritance hypothesis is disproven

`MavenDescriptor.createFromBytes(...)` (the method used to identify embedded
Bundle-ClassPath jars from their internal Maven metadata) **never parses
`pom.xml`** — it only reads `META-INF/maven/.../pom.properties`. Since
`pom.properties` always contains the full, resolved GAV regardless of whether
`pom.xml` uses `<parent>` inheritance, this is not the cause of any of the
failures reproduced here.

### #2 — verification is hardcoded against public Maven Central

Even when a correct candidate GAV is found (from `pom.properties`, from
`pom.xml`, or from a `centralsearch` lookup), it is only accepted if the exact
same artifact bytes can additionally be downloaded and verified from:

```java
// MavenDescriptor.java
public static final String MAVEN_CENTRAL_URI = "https://repo.maven.apache.org/maven2/";
```

```java
// SBOMGenerator.java, setMavenPurl(...)
var mavenArtifactBytes = contentHandler.getBinaryContent(mavenDescriptor.toArtifactURI());
...
if (equivalent(bytes, mavenArtifactBytes, differences)) {
    component.setPurl(mavenDescriptor.mavenPURL());
    return true;
}
```

If this download 404s (artifact not hosted on public Central — true for both
`jzy3d-jdt-core` on `maven.jzy3d.org` and for any internally-hosted artifact),
the whole component is silently discarded and falls back to a bare path name —
regardless of how complete and correct the embedded metadata was.

A partial mitigation already exists: `URIUtil.URIMap` / `parseRedirections(...)`
supports prefix-based URI redirection and is applied inside
`ContentHandler.getBinaryContent(...)`, activatable via the generator CLI flag
`-content-redirections "source->target"`. `tycho-sbom-plugin`'s `GeneratorMojo`
does not currently expose this flag — see [Proposed fix](#proposed-fix).

### #3 — what `centralsearch` actually changes (and what it doesn't)

`centralsearch=true` only affects **candidate discovery**: a filename-based
lookup against `search.maven.org` (`createFromJarName`) and a SHA-1 hash lookup
against `central.sonatype.com` (inside `createFromBytes`). This is what allows
`hivemq-mqtt-client` (a Gradle-built artifact with no embedded Maven metadata
at all) to be identified. It changes **nothing** about the final verification
in `setMavenPurl` described above — that remains hardcoded to public Central
regardless of `centralsearch`.

### #4 — `centralsearch` can request the wrong file extension for non-`jar` packaging

Reproduced with `com.google.guava:failureaccess:1.0.1`: its embedded
`pom.properties` is present and fully correct, and even with a matching
`-content-redirections` entry pointing at an internal mirror that has been
confirmed (via SHA-256) to hold a byte-identical copy, the component still
fails to resolve. The cause is a second, independent bug:

```java
// MavenDescriptor.java, createFromJarName(...)
return new MavenDescriptor(coordinates.getString("g"), coordinates.getString("a"),
        coordinates.getString("v"), classifier, coordinates.getString("p"));
```

`coordinates.getString("p")` is the POM **packaging** as indexed by
`search.maven.org` (`"bundle"` for `failureaccess`, since it is built with the
`maven-bundle-plugin` — a very common packaging type in the OSGi ecosystem).
This value is stored as `type` and later reused, unchanged, as the **file
extension** for the verification download:

```java
public URI toArtifactURI() {
    return toURI((classifier == null ? "" : '-' + classifier) + "." + type);
}
```

This requests `failureaccess-1.0.1.bundle` instead of
`failureaccess-1.0.1.jar` — a file that exists under that name nowhere, since
`packaging=bundle` is POM metadata, not the actual deployed file extension. The
download 404s and the component is discarded, **even though** `centralsearch`
found the fully correct GAV, and even though a content-redirection was
correctly configured. Because `createFromJarName` runs *before*
`pom.properties` is consulted, and the chain stops at the first non-null
candidate, the correct `pom.properties` data is never reached. A
`content-redirection` cannot work around this, since it only rewrites the URL
prefix, not the (wrong) trailing filename.

---

## Proposed fix

1. **`tycho-sbom-plugin`**: add a `contentRedirections` parameter to
   `GeneratorMojo` that passes `-content-redirections` through to the
   generator (currently unsupported — the Mojo builds its argument list from a
   fixed set of fields with no passthrough mechanism):

   ```java
   @Parameter(name = "content-redirections", property = "content-redirections")
   private List<String> contentRedirections;
   ```

   ```java
   if (contentRedirections != null && !contentRedirections.isEmpty()) {
       arguments.add("-content-redirections");
       for (String s : contentRedirections) {
           String trim = s.trim();
           if (!trim.isEmpty()) {
               arguments.add(trim);
           }
       }
   }
   ```

   ```xml
   <configuration>
       <contentRedirections>
           <contentRedirection>https://repo.maven.apache.org/maven2/org/jzy3d/->https://internal.example.com/artifactory/libs-release/org/jzy3d/</contentRedirection>
       </contentRedirections>
   </configuration>
   ```

   Note: redirection entries must use distinct, sufficiently specific (deep)
   source prefixes per target. `URIUtil.URIMap` is a plain map keyed by the
   literal source string — reusing the same source prefix for two different
   targets causes the second to silently replace the first.

2. **`p2repo-sbom`**: in `createFromJarName(...)`, always use `"jar"` as the
   file extension for the verification download (`toArtifactURI()`),
   independent of the POM packaging type reported by `search.maven.org`. The
   packaging value is only meaningful as a PURL qualifier (`?type=bundle`),
   never as the actual repository file extension.

3. **`p2repo-sbom`** (secondary suggestion): when the first candidate GAV
   found (via `createFromJarName`, a sibling `.pom`, or `pom.properties`)
   fails verification, fall back to the next method instead of discarding the
   component immediately. This would have produced a correct result for
   `failureaccess` even without fix #2, since its `pom.properties` alone is
   sufficient and correct.

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

### Build fails resolving p2 dependencies

**Cause:** Network issues reaching `download.eclipse.org`.

**Fix:** Ensure outbound HTTPS access to `download.eclipse.org` and Maven Central.
You can also mirror the Eclipse 2024-06 p2 repository locally.

---

### An artifact stays unresolved even though `pom.properties` is complete and correct

**Cause:** Either the artifact isn't hosted on public Maven Central (see Root
cause #2) — solvable with a `content-redirection` once available in
`tycho-sbom-plugin` — or, if a `content-redirection` is already correctly
configured and byte-identity has been confirmed (e.g. via SHA-256) and it
*still* fails, the artifact may have a non-`jar` POM packaging type (e.g.
`bundle`) and `centralsearch=true` is enabled (see Root cause #4). In that
case, disabling `centralsearch` is the only current workaround, at the cost of
losing resolution for artifacts that rely on it (e.g. Gradle-built jars with no
embedded Maven metadata).
