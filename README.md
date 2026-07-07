# tycho-sbom-demo

Minimal reproducible demo for an Eclipse Tycho SBOM-plugin resolution issue:

> **Some embedded OSGi `Bundle-ClassPath` dependencies are not resolved in SBOM
> generation when their `pom.xml` inside the JAR inherits `groupId`/`version`
> from a Maven `<parent>` instead of declaring them explicitly.**

Versions pinned for reproducibility:
| Component | Version |
|---|---|
| Tycho | **5.0.3** |
| Eclipse release repo | 2024-06 |
| `org.jzy3d:jzy3d-jdt-core` | 2.2.0 *(not resolved)* |
| `com.diogonunes:JColor` | 5.2.0 *(resolved correctly)* |

---

## Module architecture

```
tycho-sbom-demo/
├── pom.xml                              # parent – Tycho 5.0.3 + shared config
├── com.example.sbom.demo/               # OSGi eclipse-plugin bundle
│   ├── pom.xml                          # copies embedded JARs via maven-dependency-plugin
│   ├── build.properties                 # Tycho bin.includes – packs lib/ into bundle
│   └── META-INF/
│       └── MANIFEST.MF                  # Bundle-ClassPath with both embedded JARs
└── com.example.sbom.repository/        # eclipse-repository (p2 update-site)
    ├── pom.xml                          # tycho-sbom-plugin:generator configured HERE
    └── category.xml                     # includes com.example.sbom.demo bundle
```

**Why two modules?**

The `tycho-sbom-plugin:generator` goal requires a real p2 installation context
(an eclipse-repository or product). Running it on a plain `pom`-packaged parent
fails with *"One of 'installations' or 'installation' must be specified"* because
there is no built installation to analyze. The `com.example.sbom.repository`
module provides that context: it builds a p2 update-site from the demo bundle,
then the SBOM generator analyzes `target/repository/` in the `verify` phase.

`lib/jzy3d/` is **generated at build time** by `maven-dependency-plugin` and is
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

### Output locations

| Artifact | Path |
|---|---|
| OSGi bundle JAR | `com.example.sbom.demo/target/com.example.sbom.demo-1.0.0-SNAPSHOT.jar` |
| p2 repository | `com.example.sbom.repository/target/repository/` |
| Generated SBOM | `com.example.sbom.repository/target/sbom/bom.json` *(CycloneDX JSON)* |

### Expected behaviour

Both embedded dependencies appear as components in the SBOM:

```json
{ "type": "library", "group": "com.diogonunes",  "name": "JColor",         "version": "5.2.0" }
{ "type": "library", "group": "org.jzy3d",        "name": "jzy3d-jdt-core", "version": "2.2.0" }
```

### Actual behaviour

Only `JColor` appears. `jzy3d-jdt-core` is **missing** from the SBOM.

```bash
# Confirm jzy3d-jdt-core is absent (run after mvn verify):
grep -c "jzy3d-jdt-core" com.example.sbom.repository/target/sbom/bom.json \
  && echo "FOUND (unexpected)" || echo "MISSING – bug confirmed"
```

---

## Inspecting the embedded JAR metadata

Use these commands to confirm the Maven metadata layout inside each embedded JAR.
(Run `mvn verify` at least once first so `lib/` is populated.)

### JColor – works ✅

```bash
# List META-INF/maven entries
jar tf com.example.sbom.demo/lib/jzy3d/JColor-5.2.0.jar \
  | grep META-INF/maven

# Show pom.properties (contains explicit groupId, artifactId, version)
unzip -p com.example.sbom.demo/lib/jzy3d/JColor-5.2.0.jar \
  META-INF/maven/com.diogonunes/JColor/pom.properties

# Show pom.xml (contains explicit <groupId> and <version>)
unzip -p com.example.sbom.demo/lib/jzy3d/JColor-5.2.0.jar \
  META-INF/maven/com.diogonunes/JColor/pom.xml
```

### jzy3d-jdt-core – not resolved ❌

```bash
# List META-INF/maven entries
jar tf com.example.sbom.demo/lib/jzy3d/jzy3d-jdt-core-2.2.0.jar \
  | grep META-INF/maven

# Show pom.properties (contains full GAV – looks complete)
unzip -p com.example.sbom.demo/lib/jzy3d/jzy3d-jdt-core-2.2.0.jar \
  META-INF/maven/org.jzy3d/jzy3d-jdt-core/pom.properties

# Show pom.xml (NOTE: no <groupId> or <version> – both inherited from <parent>)
unzip -p com.example.sbom.demo/lib/jzy3d/jzy3d-jdt-core-2.2.0.jar \
  META-INF/maven/org.jzy3d/jzy3d-jdt-core/pom.xml
```

The key difference visible in `pom.xml`:

| | `JColor` | `jzy3d-jdt-core` |
|---|---|---|
| `<groupId>` in pom.xml | ✅ explicit | ❌ inherited from `<parent>` |
| `<version>` in pom.xml | ✅ explicit | ❌ inherited from `<parent>` |
| `groupId` in pom.properties | ✅ present | ✅ present |
| `version` in pom.properties | ✅ present | ✅ present |

### Check the generated SBOM

```bash
# Pretty-print the generated CycloneDX SBOM
cat com.example.sbom.repository/target/sbom/bom.json | python3 -m json.tool | \
  grep -A4 '"name"'

# Verify jzy3d-jdt-core is absent
grep -c "jzy3d-jdt-core" com.example.sbom.repository/target/sbom/bom.json \
  && echo "FOUND (unexpected)" || echo "MISSING (bug confirmed)"
```

---

## Root cause hypothesis

The Tycho SBOM plugin attempts to identify embedded JAR components by parsing
the `pom.xml` found under `META-INF/maven/<groupId>/<artifactId>/pom.xml` inside
the JAR. When `groupId` and/or `version` are not declared directly in that POM
but are only available via a `<parent>` element, the resolver cannot perform
parent-POM resolution (since the parent is not on the classpath) and therefore
cannot determine the full GAV. The `pom.properties` file, which **does** contain
the complete GAV, is apparently not used as a fallback – or the fallback logic
has a bug.

See [`docs/issue-body.md`](docs/issue-body.md) for a ready-to-paste Tycho bug report.

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

### `One of 'installations' or 'installation' must be specified`

**Cause:** The `generator` goal was run on a `pom`-packaged parent or on an
`eclipse-plugin` module that has no built p2 installation to analyze.

**Fix:** Run `generator` only in an `eclipse-repository` module, and set the
`<installation>` parameter to the built repository path:

```xml
<configuration>
  <installation>${project.build.directory}/repository</installation>
</configuration>
```

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

