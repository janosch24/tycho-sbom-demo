# Bug Report / Feature Request: `tycho-sbom-plugin` / `p2repo-sbom` only assign Maven identity to embedded Bundle-ClassPath JARs that are also downloadable from public Maven Central

> **This supersedes the original narrower report about `<parent>`-inherited
> `groupId`/`version` in an embedded `pom.xml`.** Systematic testing across
> three differently-sourced libraries showed that hypothesis was incomplete.
> The corrected root cause and a validated fix are below.

---

## Environment

| Item | Value |
|---|---|
| Tycho version | 5.0.3 |
| `tycho-sbom-plugin` version | 5.0.3 |
| Maven version | 3.9.x |
| Java version | 21 |
| OS | Linux x86_64 / Windows |

---

## Summary

`tycho-sbom-plugin:generator` (wrapping `eclipse-cbi/p2repo-sbom`) only ever
assigns a full Maven identity (`group`, `version`, `pkg:maven` purl) to an
embedded OSGi `Bundle-ClassPath` JAR if the exact same artifact bytes can also
be downloaded from a single, hardcoded URL:
`https://repo.maven.apache.org/maven2/`.

Correctly embedded Maven metadata (`pom.xml` and/or `pom.properties`) is
necessary but **not sufficient**. If the artifact is hosted anywhere else —
a private/internal Maven repository, or even a legitimate third-party public
Maven repository — the correctly identified GAV is silently discarded and the
component is emitted with only a raw file path, no group/version, no purl.

This affects any organization that hosts internal Maven artifacts, and any
project that embeds jars published outside Maven Central (e.g. project-run
repositories such as `maven.jzy3d.org`).

---

## Minimal reproducible example

```bash
git clone https://github.com/janosch24/tycho-sbom-demo.git
cd tycho-sbom-demo
mvn -V -e -DskipTests clean verify
```

Three embedded libraries are exercised, chosen to isolate every relevant
dimension independently:

| Library | Embeds `pom.xml`/`pom.properties`? | `pom.xml` GAV | Hosted on |
|---|---|---|---|
| `com.diogonunes:JColor:5.2.0` | both | explicit | Maven Central |
| `org.jzy3d:jzy3d-jdt-core:2.2.0` | both | via `<parent>` | `maven.jzy3d.org` (not Central) |
| `com.hivemq:hivemq-mqtt-client:1.3.16` | neither (Gradle build) | n/a | Maven Central |

---

## Result matrix

Ran with `tycho-sbom-plugin:generator`, `process-bundle-classpath=true`, in
both `centralsearch=false` and `centralsearch=true`:

| Library | pom.xml GAV form | pom.properties correct? | Hosted on | Resolved (centralsearch=false) | Resolved (centralsearch=true) |
|---|---|---|---|---|---|
| JColor | explicit | yes | central | ✅ | ✅ |
| jzy3d-jdt-core | inherited from `<parent>` | yes | other (`maven.jzy3d.org`) | ❌ | ❌ |
| hivemq-mqtt-client | n/a (no pom.xml) | n/a (no pom.properties) | central | ❌ (only `version`, no group/purl) | ✅ |

The result column correlates **only** with "hosted on", not with the
`pom.xml`/`pom.properties` columns, and not with `centralsearch`. JColor and
jzy3d-jdt-core are both correctly identified from `pom.properties` in exactly
the same way (see below); only their hosting location differs, and only that
difference is reflected in the outcome.

---

## Root cause

### 1. Embedded jars are resolved from `pom.properties`, not `pom.xml`

In [`MavenDescriptor.java`](https://github.com/eclipse-cbi/p2repo-sbom/blob/main/plugins/org.eclipse.cbi.p2repo.sbom/src/org/eclipse/cbi/p2repo/sbom/MavenDescriptor.java):

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
    if (queryCentral) {
        // SHA-1 hash lookup against central.sonatype.com — only if queryCentral
    }
    return null;
}
```

This method — used by `SBOMGenerator.gatherInnerJars(...)` for every embedded
Bundle-ClassPath jar — reads **only** `pom.properties`. The embedded `pom.xml`
is never parsed here at all (a separate `createFromPOM(byte[])` exists that
does handle `<parent>` inheritance, but it is only applied to a stray
sibling `.pom` file physically embedded next to the jar in the outer bundle,
which essentially never exists in practice — not to the jar's own internal
`META-INF/maven/.../pom.xml`).

Since Maven always writes a fully-resolved GAV into `pom.properties`
regardless of `<parent>` inheritance in `pom.xml`, this step succeeds
identically for JColor and jzy3d-jdt-core. **The originally reported
`<parent>`-inheritance problem in `pom.xml` is not actually consulted by this
code path and is not the cause of the resolution failure.**

### 2. The candidate GAV is discarded unless verified against a hardcoded Central URL

In the same file:

```java
public static final String MAVEN_CENTRAL_URI = "https://repo.maven.apache.org/maven2/";

private URI toURI(String suffix) {
    return URI.create(MAVEN_CENTRAL_URI + groupId.replace('.', '/') + "/" + artifactId + "/" + version + "/"
            + artifactId + "-" + version + suffix);
}

public URI toArtifactURI() {
    return toURI((classifier == null ? "" : '-' + classifier) + "." + type);
}
```

And in `SBOMGenerator.setMavenPurl(...)`:

```java
private boolean setMavenPurl(Component component, MavenDescriptor mavenDescriptor, byte[] bytes) {
    try {
        var mavenArtifactBytes = contentHandler.getBinaryContent(mavenDescriptor.toArtifactURI());
        getClearlyDefinedProperty(component, mavenDescriptor);
        var differences = new ArrayList<String>();
        if (equivalent(bytes, mavenArtifactBytes, differences)) {
            var purl = mavenDescriptor.mavenPURL();
            component.setPurl(purl);
            return true;
        }
        // ... otherwise recorded as a pedigree ancestor
    } catch (ContentHandler.ContentHandlerException e) {
        if (e.statusCode() != 404) {
            throw new RuntimeException(e);
        }
        // 404 is swallowed silently here
    } catch (NoSuchFileException e) {
        // same for local files
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
    return false;
}
```

And in `SBOMGenerator.createMavenJarComponent(...)`:

```java
private Component createMavenJarComponent(Component parent, String path, MavenDescriptor mavenDescriptor, byte[] bytes) {
    var component = new Component();
    component.setBomRef(parent.getBomRef() + "^" + path);
    component.setType(Component.Type.LIBRARY);
    if (setMavenPurl(component, mavenDescriptor, bytes)) {
        // If it's verified to be the identical artifact.
        component.setName(mavenDescriptor.artifactId());
        component.setGroup(mavenDescriptor.groupId());
        component.setVersion(mavenDescriptor.version());
    } else {
        // If it's got a pedigree, use the original jar path.
        component.setName(path);
    }
    return component;
}
```

A correctly identified `MavenDescriptor` is only actually applied to the
component (group/artifactId/version + purl) if `setMavenPurl` can download the
byte-/zip-identical artifact from `MAVEN_CENTRAL_URI`. A 404 (artifact not on
Central) is caught silently, `setMavenPurl` returns `false`, and
`createMavenJarComponent` falls back to a bare, path-named component — with no
indication in logs or output of *why* it was not resolved.

### 3. `centralsearch` does not affect this verification step

`centralsearch` (`-central-search`) only changes how a **candidate** GAV is
found for jars with no/incomplete embedded metadata:

- `MavenDescriptor.createFromJarName(...)`, tried first in
  `gatherInnerJars(...)`, parses the jar's filename and queries
  `search.maven.org` — only when `centralsearch=true`. This is what lets
  `hivemq-mqtt-client` (a Gradle-built jar with no `META-INF/maven` at all)
  get identified.
- The SHA-1 hash lookup against `central.sonatype.com` inside
  `createFromBytes(...)` — same restriction.

Both of these still funnel into the same `setMavenPurl` verification against
`MAVEN_CENTRAL_URI`. `centralsearch=true` therefore fixes `hivemq-mqtt-client`
(genuinely hosted on Central) but has **no effect at all** on
`jzy3d-jdt-core` (hosted on `maven.jzy3d.org`), confirmed by direct testing.

---

## Proposed fix

`p2repo-sbom` already has infrastructure for exactly this situation:
[`URIUtil.java`](https://github.com/eclipse-cbi/p2repo-sbom/blob/main/plugins/org.eclipse.cbi.p2repo.sbom/src/org/eclipse/cbi/p2repo/sbom/URIUtil.java):

```java
public static URIMap parseRedirections(List<String> redirections) {
    var uriRedirections = new URIMap();
    for (var uriRedirection : redirections) {
        var pair = uriRedirection.split("->");
        if (pair.length != 2) {
            throw new IllegalArgumentException("Expected a '->' in the redirection:" + uriRedirection);
        }
        uriRedirections.put(toURI(pair[0]), toURI(pair[1]));
    }
    return uriRedirections;
}
```

wired into `ContentHandler` in `SBOMGenerator.java`:

```java
contentHandler = new ContentHandler(getArgument("-cache", args, null),
        parseRedirections(getArguments("-content-redirections", args, List.of())),
        ...);
```

and applied on every content fetch, including the one `setMavenPurl` performs:

```java
public byte[] getBinaryContent(URI uri) throws IOException {
    return getContent(uriMap.redirect(uri), Files::readAllBytes, Files::write, BodyHandlers.ofByteArray());
}
```

**The generator already supports redirecting the Central verification lookup
to an arbitrary repository — `tycho-sbom-plugin`'s `GeneratorMojo` just does
not expose it.**

### Patch to `tycho-sbom-plugin/src/main/java/org/eclipse/tycho/sbom/plugin/GeneratorMojo.java`

```java
/**
 * A list of URI redirections in the form "source->target" (both ending in
 * "/") applied to all content lookups performed by the generator — most
 * notably the Maven Central artifact verification in setMavenPurl. Useful
 * when embedded artifacts are hosted on an internal or third-party Maven
 * repository rather than on Maven Central itself.
 */
@Parameter(name = "content-redirections", property = "content-redirections")
private List<String> contentRedirections;
```

and, in `execute()`, analogous to the existing `p2sources` handling:

```java
if (contentRedirections != null && !contentRedirections.isEmpty()) {
    arguments.add("-content-redirections");
    for (String s : contentRedirections) {
        String trim = s.trim();
        if (trim.isEmpty()) {
            continue;
        }
        arguments.add(trim);
    }
}
```

### Validated configuration (tested against this demo repo)

Redirection rules must use **specific path prefixes** per target, not the
generic `.../maven2/` prefix repeated with different targets — the underlying
`URIMap` is a plain `source → target` map keyed by the literal source string,
so a second rule with the *same* source silently replaces the first one
instead of adding a second route:

```xml
<plugin>
    <groupId>org.eclipse.tycho</groupId>
    <artifactId>tycho-sbom-plugin</artifactId>
    <executions>
        <execution>
            <id>generate-sbom</id>
            <goals>
                <goal>generator</goal>
            </goals>
            <configuration>
                <installation>${project.build.directory}/products/com.example.sbom.demo.product/linux/gtk/x86_64</installation>
                <process-bundle-classpath>true</process-bundle-classpath>
                <content-redirections>
                    <content-redirection>https://repo.maven.apache.org/maven2/org/jzy3d/-&gt;https://maven.jzy3d.org/releases/org/jzy3d/</content-redirection>
                </content-redirections>
            </configuration>
        </execution>
    </executions>
</plugin>
```

With this configuration (tested end-to-end against a locally patched
`tycho-sbom-plugin`), both `jzy3d-jdt-core` and an internal, non-public
artifact were resolved with correct `group`/`version`/`pkg:maven` purl, with
no change required in `p2repo-sbom` itself.

### Secondary suggestion for `p2repo-sbom`

Independent of the above, it may be worth softening `setMavenPurl`'s
all-or-nothing behaviour: when a candidate GAV is available from
`pom.properties` but Central verification fails (404), consider keeping
`group`/`artifactId`/`version` on the component with a lower-confidence marker
(e.g. an unverified `pkg:maven` purl, or an explicit property/note), rather
than discarding the identity entirely and falling back to a bare file path.
This would make partially-trusted results visible in the SBOM instead of
silently indistinguishable from "no metadata found at all".

---

## Additional context

- All three embedded JARs are unmodified, verbatim copies of the original,
  published artifacts (copied via `maven-dependency-plugin`), confirmed
  byte-identical via SHA-256 hash comparison between the standalone artifact
  and the bytes actually processed by the generator inside the built bundle.
- This is not specific to any one library — any embedded jar hosted outside
  Maven Central is affected, independent of how well-formed its embedded
  Maven metadata is.
- Related upstream issue on embedded-jar support in general:
  https://github.com/eclipse-cbi/p2repo-sbom/issues/7
