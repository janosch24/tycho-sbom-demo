# tycho-sbom-demo

Minimal reproducible demo for an Eclipse Tycho SBOM-plugin resolution issue:

> **Some embedded OSGi `Bundle-ClassPath` dependencies are not correctly resolved in SBOM
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

The `package` phase builds the p2 update-site **and** materializes the Eclipse
product (creating `configuration/config.ini`). The `verify` phase then runs the
SBOM generator against the installed product.

### Verify the installation exists and contains `config.ini`

After `mvn verify` completes, confirm the installation was created:

```bash
# Check config.ini exists in the materialized product
ls com.example.sbom.repository/target/products/com.example.sbom.demo.product/linux/gtk/x86_64/configuration/config.ini

# Peek at the osgi.bundles entry
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

Both embedded dependencies appear as components containing group, name and version in the SBOM:

```json
{ "type": "library", "group": "com.diogonunes",  "name": "JColor",         "version": "5.2.0" }
{ "type": "library", "group": "org.jzy3d",        "name": "jzy3d-jdt-core", "version": "2.2.0" }
```

### Actual behaviour

Only `JColor` appears. `jzy3d-jdt-core` is **missing** from the SBOM.

```bash
# Confirm jzy3d-jdt-core is absent (run after mvn verify):
grep -c "jzy3d-jdt-core" com.example.sbom.repository/target/com.example.sbom.repository.json \
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
cat com.example.sbom.repository/target/com.example.sbom.repository.json | python3 -m json.tool | \
  grep -A4 '"name"'

# Verify jzy3d-jdt-core is absent
grep -c "jzy3d-jdt-core" com.example.sbom.repository/target/com.example.sbom.repository.json \
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

