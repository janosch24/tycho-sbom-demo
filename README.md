# tycho-sbom-demo

Minimal reproducible demo for an Eclipse Tycho SBOM-plugin resolution issue:

> **Some embedded OSGi `Bundle-ClassPath` dependencies are not resolved in SBOM
> generation when their `pom.xml` inside the JAR inherits `groupId`/`version`
> from a Maven `<parent>` instead of declaring them explicitly.**

Versions pinned for reproducibility:
| Component | Version |
|---|---|
| Tycho | **5.0.2** |
| Eclipse release repo | 2024-06 |
| `org.jzy3d:jzy3d-jdt-core` | 2.2.0 *(not resolved)* |
| `com.diogonunes:JColor` | 5.2.0 *(resolved correctly)* |

---

## Project layout

```
tycho-sbom-demo/
├── pom.xml                          # parent – Tycho 5.0.2 + SBOM plugin
└── com.example.sbom.demo/
    ├── pom.xml                      # eclipse-plugin module; copies embedded JARs
    ├── build.properties             # Tycho bin.includes – packs lib/ into bundle
    └── META-INF/
        └── MANIFEST.MF              # Bundle-ClassPath with both embedded JARs
```

`lib/jzy3d/` is **generated at build time** by `maven-dependency-plugin` and is
not committed to version control.

---

## Reproducing the SBOM discrepancy

### Prerequisites

- Java 17+
- Maven 3.9+

### Run the build

```bash
mvn -V -e -DskipTests clean verify
```

After a successful build the plugin jar is at:

```
com.example.sbom.demo/target/com.example.sbom.demo-1.0.0-SNAPSHOT.jar
```

The SBOM (CycloneDX JSON) is written to:

```
com.example.sbom.demo/target/bom.json
```

### Expected behaviour

Both embedded dependencies appear as components in the SBOM:

```json
{ "type": "library", "group": "com.diogonunes",  "name": "JColor",         "version": "5.2.0" }
{ "type": "library", "group": "org.jzy3d",        "name": "jzy3d-jdt-core", "version": "2.2.0" }
```

### Actual behaviour

Only `JColor` appears. `jzy3d-jdt-core` is **missing** from the SBOM.

---

## Inspecting the embedded JAR metadata

Use these commands to confirm the Maven metadata layout inside each embedded JAR.

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
cat com.example.sbom.demo/target/bom.json | python3 -m json.tool | \
  grep -A4 '"name"'

# Verify jzy3d-jdt-core is absent
grep -c "jzy3d-jdt-core" com.example.sbom.demo/target/bom.json \
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
