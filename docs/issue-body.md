# Bug Report: `tycho-sbom-plugin` / `p2repo-sbom` frequently fail to assign Maven identity to embedded Bundle-ClassPath JARs, for several independent reasons

> **This supersedes the original narrower report about `<parent>`-inherited
> `groupId`/`version` in an embedded `pom.xml`.** Systematic testing across
> four differently-sourced libraries showed that hypothesis was wrong, and
> uncovered two distinct, code-verified bugs instead.
>
> **Paste this text into a new Eclipse Tycho GitHub issue.**
> Repository: https://github.com/eclipse-tycho/tycho

---

## Environment

| Item | Value |
|---|---|
| Tycho version | 5.0.3 |
| `tycho-sbom-plugin` version | 5.0.3 |
| `p2repo-sbom` (generator) | nightly build used by Tycho 5.0.3 |
| Maven version | 3.9.x |
| Java version | 21 |
| OS | Windows / Linux x86_64 |

---

## Summary

`tycho-sbom-plugin` (goal `generator`, `process-bundle-classpath=true`) is
supposed to identify embedded OSGi `Bundle-ClassPath` JARs as proper Maven
components (`group`, `name`, `version`, `purl`) in the generated CycloneDX
SBOM. In practice this frequently fails — not because the embedded metadata
(`pom.properties` / `pom.xml`) is incomplete or incorrect, but because of two
independent issues in the underlying `p2repo-sbom` generator:

1. A correctly identified candidate GAV is only ever **accepted** if the
   exact same artifact bytes can additionally be downloaded and verified from
   a hardcoded public Maven Central URL. Artifacts hosted only on third-party
   or internal repositories are discarded regardless of how correct their
   embedded metadata is.
2. When `centralsearch` is enabled and an artifact has a non-`jar` POM
   packaging type (e.g. `bundle`, common for OSGi-oriented libraries built
   with the `maven-bundle-plugin`), the generator constructs the verification
   download URL using that packaging type as the **file extension**, even
   though the actual deployed file is always a plain `.jar`. This causes a
   404 and the component is discarded — even when a fully correct
   `pom.properties` is present and even when a correct repository
   redirection has been configured.

This demo reproduces both issues with four real-world libraries, chosen to
isolate each variable (embedded metadata quality, hosting location, POM
packaging type).

---

## Minimal reproducible example

```bash
git clone https://github.com/janosch24/tycho-sbom-demo.git
cd tycho-sbom-demo
mvn -V -e -DskipTests clean verify
```

The `package` phase builds the p2 update-site **and** materializes an Eclipse
product installation (required by the generator — see *Steps to reproduce*).
The `verify` phase runs the SBOM generator against the installed product.

Inspect the result:

```bash
cat com.example.sbom.repository/target/com.example.sbom.repository.json | python3 -m json.tool
```

---

## Steps to reproduce

> **Important:** `tycho-sbom-plugin:generator` requires an *installed* Eclipse
> product layout (with `configuration/config.ini`), not a plain p2 update-site.
> A plain `eclipse-repository` update-site (`target/repository/`) does **not**
> contain `configuration/`, so the generator throws `NoSuchFileException`.
> The steps below use a `.product` file to trigger product materialization via
> `tycho-p2-director-plugin:materialize-products`.

1. Create an OSGi `eclipse-plugin` bundle.
2. Embed four JARs under `lib/` and list them on `Bundle-ClassPath` in
   `MANIFEST.MF`:
   - `com.diogonunes:JColor:5.2.0` (Maven Central, flat `pom.xml`)
   - `org.jzy3d:jzy3d-jdt-core:2.2.0` (third-party repo `maven.jzy3d.org`,
     `pom.xml` inherits GAV from `<parent>`)
   - `com.hivemq:hivemq-mqtt-client:1.3.16` (Maven Central, Gradle build — no
     embedded Maven metadata at all)
   - `com.google.guava:failureaccess:1.0.1` (Maven Central, POM packaging
     `bundle`)
3. Build the bundle and include it in an `eclipse-repository` p2 update-site.
4. Add a minimal `.product` file to the `eclipse-repository` module referencing
   the bundle and `org.eclipse.osgi`. Tycho will automatically run
   `materialize-products` during the `package` phase, creating a real Eclipse
   installation directory under `target/products/<uid>/<os>/<ws>/<arch>/`.
5. Configure `tycho-sbom-plugin:generator` on the repository module:
   ```xml
   <configuration>
     <installation>${project.build.directory}/products/com.example.sbom.demo.product/linux/gtk/x86_64</installation>
     <process-bundle-classpath>true</process-bundle-classpath>
     <centralsearch>true</centralsearch>
   </configuration>
   ```
6. Run `mvn verify` and inspect
   `com.example.sbom.repository/target/com.example.sbom.repository.json`.

---

## Expected behaviour

All four embedded dependencies appear as fully resolved Maven components:

```json
{ "type": "library", "group": "com.diogonunes",   "name": "JColor",            "version": "5.2.0",  "purl": "pkg:maven/com.diogonunes/JColor@5.2.0" }
{ "type": "library", "group": "org.jzy3d",         "name": "jzy3d-jdt-core",    "version": "2.2.0",  "purl": "pkg:maven/org.jzy3d/jzy3d-jdt-core@2.2.0" }
{ "type": "library", "group": "com.hivemq",        "name": "hivemq-mqtt-client","version": "1.3.16", "purl": "pkg:maven/com.hivemq/hivemq-mqtt-client@1.3.16" }
{ "type": "library", "group": "com.google.guava",  "name": "failureaccess",     "version": "1.0.1",  "purl": "pkg:maven/com.google.guava/failureaccess@1.0.1" }
```

## Actual behaviour

| Library | `group`/`purl` set? | Notes |
|---|---|---|
| JColor 5.2.0 | ✅ yes | resolves correctly |
| jzy3d-jdt-core 2.2.0 | ❌ no | hosted on `maven.jzy3d.org`, not public Central |
| hivemq-mqtt-client 1.3.16 | ❌ no (without `centralsearch`) / ✅ yes (with `centralsearch=true`) | no embedded Maven metadata |
| failureaccess 1.0.1 | ❌ no | wrong file extension requested (see Root cause #2 below) |

Example of an unresolved entry (falls back to internal JAR path):

```json
{
  "type": "library",
  "bom-ref": "plugins/com.example.sbom.demo_1.0.0....jar^lib/jzy3d-jdt-core-2.2.0.jar",
  "name": "lib/jzy3d-jdt-core-2.2.0.jar"
}
```

---

## Result matrix

| # | Library | `pom.xml` embedded | `pom.xml` GAV | `pom.properties` embedded | `pom.properties` GAV correct | Hosted on | Result |
|---|---|---|---|---|---|---|---|
| 1 | JColor 5.2.0 | yes | flat | yes | yes | Maven Central | ok |
| 2 | jzy3d-jdt-core 2.2.0 | yes | inherited from `<parent>` | yes | yes | `maven.jzy3d.org` (third-party) | not ok |
| 3 | hivemq-mqtt-client 1.3.16 | no | n/a | no | n/a | Maven Central | not ok (ok with `centralsearch=true`) |
| 4 | failureaccess 1.0.1 | no | n/a | yes | yes | Maven Central (+ confirmed byte-identical internal mirror + matching `content-redirection`) | not ok |

**Key observation:** the result correlates with *where the artifact is hosted*
and *how its candidate GAV was found* (rows 1–3), and with *POM packaging type*
combined with `centralsearch` (row 4) — never with whether `pom.xml` /
`pom.properties` are present or correct. Rows 1 and 2 differ exactly in
`<parent>`-inheritance in `pom.xml`, yet both resolve identically well from
`pom.properties` — disproving the original hypothesis of this issue.

---

## Root cause #1 — `pom.xml` is never parsed for embedded jars; `<parent>`-inheritance is not the problem

`MavenDescriptor.createFromBytes(...)` (used to identify embedded
Bundle-ClassPath jars) only reads `META-INF/maven/.../pom.properties`:

```java
public static MavenDescriptor createFromBytes(byte[] bytes, boolean queryCentral, ContentHandler contentHandler) {
    try (var stream = new JarInputStream(new ByteArrayInputStream(bytes))) {
        ZipEntry entry;
        while ((entry = stream.getNextEntry()) != null) {
            var name = entry.getName();
            if (name.startsWith("META-INF/maven/") && name.endsWith("pom.properties")) {
                var properties = new Properties();
                properties.load(stream);
                var artifactId = properties.getProperty("artifactId");
                var groupId = properties.getProperty("groupId");
                var version = properties.getProperty("version");
                if (artifactId != null && groupId != null && version != null) {
                    return new MavenDescriptor(groupId, artifactId, version, null, "jar");
                }
            }
        }
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
    ...
}
```

`pom.xml` is never parsed here. Since `pom.properties` always contains the
full resolved GAV, `<parent>`-inherited `groupId`/`version` in `pom.xml` (the
original hypothesis of this issue) has no effect on resolution.

## Root cause #2 — verification is hardcoded against public Maven Central

Even a correctly found candidate GAV is only *accepted* if byte-identical to
a download from a hardcoded constant:

```java
// MavenDescriptor.java
public static final String MAVEN_CENTRAL_URI = "https://repo.maven.apache.org/maven2/";

private URI toURI(String suffix) {
    return URI.create(MAVEN_CENTRAL_URI + groupId.replace('.', '/') + "/" + artifactId + "/" + version + "/"
            + artifactId + "-" + version + suffix);
}
```

```java
// SBOMGenerator.java
var mavenArtifactBytes = contentHandler.getBinaryContent(mavenDescriptor.toArtifactURI());
...
if (equivalent(bytes, mavenArtifactBytes, differences)) {
    component.setPurl(mavenDescriptor.mavenPURL());
    return true;
}
```

If this download fails (404 — artifact not hosted on public Central), the
component is silently discarded and falls back to a bare path name,
regardless of embedded-metadata quality. This affects both legitimate
third-party repositories (`jzy3d-jdt-core` on `maven.jzy3d.org`) and internal
company repositories equally.

A partial mitigation already exists: `URIUtil.URIMap` / `parseRedirections(...)`
supports prefix-based URI redirection and is applied in
`ContentHandler.getBinaryContent(...)`, activatable via the generator CLI flag
`-content-redirections "source->target"`. `GeneratorMojo` in
`tycho-sbom-plugin` does not currently expose this flag at all — see
*Proposed fix* below.

## What `centralsearch` actually changes (and what it doesn't)

`centralsearch=true` only affects **candidate discovery**: a filename-based
lookup against `search.maven.org` (`createFromJarName`) and a SHA-1 hash
lookup against `central.sonatype.com` (inside `createFromBytes`). This is
what allows `hivemq-mqtt-client` (Gradle-built, no embedded Maven metadata at
all) to be identified. It changes **nothing** about the final verification
in `setMavenPurl` above, which always targets public Central regardless of
`centralsearch`.

## Root cause #3 — `centralsearch` can request the wrong file extension for non-`jar` packaging

Reproduced with `com.google.guava:failureaccess:1.0.1`. Its embedded
`pom.properties` is present and fully correct. Even with a matching
`-content-redirections` entry pointing at an internal mirror confirmed (via
SHA-256) to hold a byte-identical copy of the artifact, the component still
fails to resolve.

```java
// MavenDescriptor.java, createFromJarName(...)
var queryResult = contentHandler.getContent(URI.create(query)); // search.maven.org solr query
...
return new MavenDescriptor(coordinates.getString("g"), coordinates.getString("a"),
        coordinates.getString("v"), classifier, coordinates.getString("p"));
```

```java
public URI toArtifactURI() {
    return toURI((classifier == null ? "" : '-' + classifier) + "." + type);
}
```

`coordinates.getString("p")` is the POM **packaging** indexed by
`search.maven.org` — `"bundle"` for `failureaccess`, since it's built with the
`maven-bundle-plugin` (a very common OSGi packaging convention). This value is
stored in `type` and reused, unchanged, as the **file extension** for the
verification download. The generator therefore requests
`failureaccess-1.0.1.bundle`, a file that exists under that name nowhere
(`packaging=bundle` is POM metadata, not the deployed file's actual
extension — the physical file on Central is `failureaccess-1.0.1.jar`). The
download 404s and the component is discarded.

Crucially, `createFromJarName` (triggered by `centralsearch=true`) runs
*before* `pom.properties` is consulted for embedded jars, and the resolution
chain stops at the first non-null candidate. So the correct
`pom.properties` GAV for `failureaccess` is never even reached — it is
pre-empted by a candidate with the wrong extension from a completely
different resolution method. A `content-redirection` cannot compensate for
this, since it only rewrites the URL path prefix, not the (wrong) trailing
filename.

Verified independently, outside of the plugin, that the embedded jar, the
internal mirror copy, and the public Maven Central copy are all byte-for-byte
identical (SHA-256), ruling out any content/repackaging discrepancy as the
cause.

---

## Proposed fix

1. **`tycho-sbom-plugin`**: expose the existing `-content-redirections`
   generator flag on `GeneratorMojo`, currently missing entirely:

   ```java
   /**
    * A list of URI redirections in the form "source->target" applied to all
    * content lookups (e.g. Maven Central artifact verification), useful when
    * artifacts are hosted on an internal/mirrored repository.
    */
   @Parameter(name = "content-redirections", property = "content-redirections")
   private List<String> contentRedirections;
   ```

   and in `execute()`, analogous to the existing `p2sources` handling:

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

   Example resulting `pom.xml` configuration:

   ```xml
   <configuration>
       <contentRedirections>
           <contentRedirection>https://repo.maven.apache.org/maven2/org/jzy3d/->https://internal.example.com/artifactory/libs-release/org/jzy3d/</contentRedirection>
       </contentRedirections>
   </configuration>
   ```

   Note for anyone implementing this: redirection sources must be distinct
   and sufficiently specific per target. `URIUtil.URIMap` is a plain map
   keyed by the literal source string; reusing the same source prefix for two
   different targets causes the second configured redirection to silently
   replace the first for that key.

2. **`p2repo-sbom`**: in `createFromJarName(...)`, always use `"jar"` as the
   file extension when building the verification download URI
   (`toArtifactURI()`), independent of the POM packaging type reported by
   `search.maven.org`. The packaging value is only meaningful as a PURL
   qualifier (`?type=bundle`), never as the actual repository file extension.

3. **`p2repo-sbom`** (secondary suggestion): consider not discarding a
   component outright when the *first* candidate GAV found (via
   `createFromJarName`, a sibling `.pom`, or `pom.properties`, in that order)
   fails verification — instead fall back to the next method. This alone
   would have produced a correct result for `failureaccess`, since its
   `pom.properties` is sufficient and correct, independent of fix #2.

4. **`p2repo-sbom`** (secondary suggestion, from root cause #2): when
   verification against Central 404s and no redirection matches, consider
   retaining the GAV with a lower-confidence marker (e.g. "unverified")
   instead of discarding all Maven identity information.

---

## Additional context

- All four libraries are unmodified, publicly published artifacts (or, for
  `failureaccess`, additionally confirmed byte-identical to an internal
  mirror copy via SHA-256). None of the embedded metadata was manually
  altered.
- This affects any project embedding third-party-hosted, internally-hosted,
  Gradle-built, or `maven-bundle-plugin`-packaged dependencies via OSGi
  `Bundle-ClassPath` — all common, unremarkable patterns in the wider Java/OSGi
  ecosystem.
- Demo repository with full reproduction steps, build logs and SBOM output
  for all four cases: https://github.com/janosch24/tycho-sbom-demo
- Related issue (embedded-jar support in general):
  https://github.com/eclipse-cbi/p2repo-sbom/issues/7
