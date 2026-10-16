---
name: minecraft-plugin
description: Create a ready-to-build Minecraft server plugin project in Java (Spigot/Bukkit/Paper API) with Maven or Gradle, for any Minecraft version from 1.8 up to the latest (including the 2026 "26.x" versions). Interviews the user for MC version, Java version, and build tool, decides single-module vs multi-module, scaffolds the project (main class extends JavaPlugin, prints "enabled"/"disabled", with getting-started comments), builds and verifies it before delivering. Use this skill whenever the user wants to make, start, scaffold, or set up a Minecraft plugin, Spigot/Paper/Bukkit plugin, or server mod in Java, even if they only say "make me a plugin for my MC server" or don't mention Maven/Gradle.
---

# Minecraft Plugin Creator

Goal: hand the user a plugin project that **builds on the first try**, targets the right Minecraft and Java versions, and is friendly to a beginner. The three ways this goes wrong are a wrong Java target (server refuses the jar with "Unsupported class file major version"), a wrong API version string (dependency doesn't resolve), and delivering something that was never compiled. The steps below exist to prevent those.

## Step 1 — Interview (one question at a time, in this order)

Skip any question the user already answered in their request. Use AskUserQuestion when available (the user can always type their own answer), otherwise ask in plain text. Keep wording simple; many users are not programmers.

**1. Minecraft version** (supported range: 1.8 → latest).
Look up the current latest release first (WebSearch "latest Minecraft Java Edition release"), because it changes and your memory is stale. Offer the latest plus 2–3 popular picks (e.g. latest, 1.21.x, 1.20.4, 1.16.5/1.12.2, 1.8.8) and accept anything typed. Mojang switched to year-based numbering in 2026: after 1.21.11 came **26.1, 26.2, 26.3 …** (as of Oct 2026 the latest is 26.3; use that as the fallback if the search returns nothing useful, and say you couldn't confirm it). Reject anything below 1.8.

**2. Java version.** Show only the versions that work for the chosen MC version (table below) and mark the lowest as the default. If the user answers "nothing / don't care / you choose", use the lowest. Why lowest: a plugin compiled for a newer Java refuses to load on servers running older Java, so the lowest target is the most compatible.

**3. Build tool: Maven or Gradle.** Always ask; do not assume. If they don't know, explain in one line each (Maven = XML `pom.xml`, the most common in the plugin world; Gradle = `build.gradle.kts` script, faster incremental builds) and let them pick.

**4. Single vs multi-module** — you decide, using the scoring below. Ask only when unsure.

Defaults for everything else (state them in the final summary, don't ask): plugin name = what the user called it (else `MyPlugin`), group `com.example` (use their domain if given), version `1.0.0`, main class named after the plugin in PascalCase (e.g. `ShopHelper`), API = Spigot API for MC ≤ 1.21.11 (also runs on Paper/Purpur) and Paper API for 26.x.

### MC → Java table

| Minecraft | Minimum Java (default) | Other sensible targets |
|---|---|---|
| 1.8 – 1.11.x | 8 | — (servers of this era only run on 8) |
| 1.12 – 1.16.x | 8 | 11 |
| 1.17.x | 16 | 17 |
| 1.18 – 1.20.4 | 17 | 21 |
| 1.20.5 – 1.21.x | 21 | 25 |
| 26.x (2026+) | 25 | — |

If a version isn't in the table (a future 27.x, say), check docs.papermc.io for the Java requirement instead of guessing; a one-minute lookup beats a broken jar.

## Step 2 — Single or multi-module (the "calculation")

Add one point for each that is true:

- Supports **several MC versions** with version-specific code (NMS/adapters, or API differences) → needs one module per version family
- Needs a **separate public API** other plugins will depend on
- Needs a **shared core** reused by more than one platform (Bukkit + BungeeCord/Velocity) or a standalone tool/library
- Large codebase with clearly separate subsystems or several developers
- The user explicitly mentions modules, API, or multiple platforms

Score 0 → **single module**. Score ≥ 2 → **multi-module**. Score 1 → ask the user one short question. A beginner asking for "a plugin" is almost always single-module; multi-module adds build complexity that only pays off in the cases above, so don't over-engineer.

Multi-module layout used by this skill: `core` (pure Java, no Minecraft dependency, easy to unit test) + `plugin` (depends on `core`, contains the `JavaPlugin` main class, produces the one jar that is bundled with `core` inside). More modules (`api`, `nms-v1_21`, …) follow the same pattern: add a directory, register it in the parent, depend on it from `plugin`.

## Step 3 — Pick the dependency coordinates

**Normalize the version.** Spigot's API is published per patch, so bare versions map to the last patch: 1.8→1.8.8, 1.9→1.9.4, 1.10→1.10.2, 1.11→1.11.2, 1.12→1.12.2, 1.13→1.13.2, 1.14→1.14.4, 1.15→1.15.2, 1.16→1.16.5. Others are used as given (`1.21.4`).

| API | Repository | Coordinates |
|---|---|---|
| Spigot (MC ≤ 1.21.11) | `https://hub.spigotmc.org/nexus/content/repositories/snapshots/` and `https://oss.sonatype.org/content/groups/public/` (needed for transitive `bungeecord-chat`) | `org.spigotmc:spigot-api:<mc>-R0.1-SNAPSHOT` |
| Paper (any version, required for 26.x) | `https://repo.papermc.io/repository/maven-public/` | ≤ 1.21.11: `io.papermc.paper:paper-api:<mc>-R0.1-SNAPSHOT`; 26.x: `io.papermc.paper:paper-api:<mc>.build.+` in Gradle, `[<mc>.build,)` in Maven (e.g. `26.3.build.+`) |

The dependency is **`compileOnly` (Gradle) / `provided` (Maven)**: the server supplies the API at runtime, and bundling it would bloat or break the jar.

**plugin.yml `api-version`:** omit for MC < 1.13 (the field doesn't exist there). For 1.13+ write `'1.<minor>'` (e.g. `'1.21'`), for 26.x write `'26.<n>'` (e.g. `'26.3'`). Always quote it. A server older than `api-version` refuses to load the plugin, so don't set it higher than the chosen MC version.

## Step 4 — Scaffold

Use this layout (single module). Package = `<group>.<lowercase name without spaces>`; artifact id = lowercase-hyphenated name.

```
<name>/
├── pom.xml  |  build.gradle.kts + settings.gradle.kts (+ gradle wrapper)
├── README.md            (how to build, install, run — see below)
├── .gitignore           (target/, build/, .gradle/, .idea/, *.iml, out/)
└── src/main/
    ├── java/<package path>/<Main>.java
    └── resources/plugin.yml
```

### Main class (always)

The main class **must extend `JavaPlugin`**, print `enabled` in `onEnable` and `disabled` in `onDisable` with `System.out.println` (the user asked for sout specifically; mention in a comment that `getLogger()` is the idiomatic choice once they're comfortable), and carry beginner comments explaining how to get started. Keep the code Java-8-compatible regardless of target (no `var`, no records) so one template fits every version. Template:

```java
package @@PACKAGE@@;

import org.bukkit.plugin.java.JavaPlugin;

/**
 * @@NAME@@ — main plugin class. The server finds it through the "main:" line in plugin.yml.
 *
 * GETTING STARTED
 *  1. Build the jar:   @@BUILD_CMD@@
 *  2. Copy @@JAR_PATH@@ into your server's "plugins" folder.
 *  3. Start (or /reload) the server and look for "enabled" in the console.
 *
 * WHERE TO ADD YOUR CODE
 *  - onEnable():  runs once when the plugin loads. Register listeners and commands here.
 *  - onDisable(): runs when the server stops/reloads. Save data and clean up here.
 */
public class @@MAIN@@ extends JavaPlugin {

    @Override
    public void onEnable() {
        // Beginner-friendly console message. (Later, prefer getLogger().info("enabled").)
        System.out.println("enabled");

        // --- Ideas for what to put here next (uncomment and adapt) ---
        // Listen to events:  getServer().getPluginManager().registerEvents(new MyListener(), this);
        // Add a command:     getCommand("hello").setExecutor(new HelloCommand());
        //                    (the command must also be declared under "commands:" in plugin.yml)
        // Create config.yml: saveDefaultConfig();   // after adding src/main/resources/config.yml
    }

    @Override
    public void onDisable() {
        System.out.println("disabled");
    }
}
```

### plugin.yml

```yaml
name: @@NAME@@
version: '@@VERSION_PLACEHOLDER@@'   # Maven: ${project.version}  |  Gradle: ${version}  (filled in at build time)
main: @@PACKAGE@@.@@MAIN@@
api-version: '@@API_VERSION@@'      # omit this line for MC < 1.13
description: @@NAME@@ plugin
author: @@AUTHOR@@
```

Resource filtering is a classic trap: if it isn't enabled, the jar ships with a literal `${project.version}` and the server logs a bad-version warning. Enable it only for `plugin.yml` so other resources (e.g. a config.yml containing `${...}`) aren't mangled.

### Maven — single module `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>@@GROUP@@</groupId>
  <artifactId>@@ARTIFACT@@</artifactId>
  <version>1.0.0</version>
  <name>@@NAME@@</name>

  <properties>
    <maven.compiler.release>@@JAVA@@</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <repositories><!-- the repos for the chosen API, from the table in Step 3 --></repositories>

  <dependencies>
    <dependency>
      <groupId>…</groupId><artifactId>…</artifactId><version>…</version>
      <scope>provided</scope>
    </dependency>
  </dependencies>

  <build>
    <finalName>${project.name}-${project.version}</finalName>
    <resources>
      <resource><directory>src/main/resources</directory><filtering>true</filtering>
        <includes><include>plugin.yml</include></includes></resource>
      <resource><directory>src/main/resources</directory><filtering>false</filtering>
        <excludes><exclude>plugin.yml</exclude></excludes></resource>
    </resources>
    <plugins>
      <plugin><artifactId>maven-compiler-plugin</artifactId><version>3.13.0</version></plugin>
      <plugin><artifactId>maven-resources-plugin</artifactId><version>3.3.1</version></plugin>
      <plugin><artifactId>maven-jar-plugin</artifactId><version>3.4.2</version></plugin>
    </plugins>
  </build>
</project>
```

`maven.compiler.release` (not `source`/`target`) is used because it also checks you only call APIs that exist in that Java version. Build: `mvn clean package` → `target/<Name>-1.0.0.jar`.

### Gradle — single module (Kotlin DSL)

`settings.gradle.kts`: `rootProject.name = "@@ARTIFACT@@"`

`build.gradle.kts`:

```kotlin
plugins { java }

group = "@@GROUP@@"
version = "1.0.0"

repositories {
    mavenCentral()
    // the repos for the chosen API, from the table in Step 3
}

dependencies {
    compileOnly("<coordinates>")
}

tasks.withType<JavaCompile>().configureEach {
    options.encoding = "UTF-8"
    options.release.set(@@JAVA@@)
}

tasks.processResources {
    filesMatching("plugin.yml") { expand("version" to project.version) }
}

tasks.jar { archiveBaseName.set("@@NAME@@") }
```

Don't use a `java.toolchain` block: it triggers JDK auto-download, which fails on offline machines; `options.release` gives the same bytecode target with whatever JDK is installed (it must be ≥ the target). Generate the wrapper with `gradle wrapper` when Gradle is installed (pick a Gradle that supports the JDK: Java 25 needs Gradle 9.1+). Build: `gradle build` (or `./gradlew build`) → `build/libs/<Name>-1.0.0.jar`.

### Multi-module deltas

Both tools: `core/src/main/java/<package>/core/CoreInfo.java` (a tiny pure-Java class so the module isn't empty) and `plugin/src/main/...` holding the main class and `plugin.yml`. The final jar must contain the `core` classes, because the server only loads one jar.

**Maven:** parent `pom.xml` with `<packaging>pom</packaging>`, `<modules><module>core</module><module>plugin</module></modules>`, the shared `<properties>`, and `pluginManagement` for the three plugins above. `core/pom.xml` is a plain jar with the parent reference. `plugin/pom.xml` has the API dependency (provided), a compile dependency on `${project.groupId}:@@ARTIFACT@@-core:${project.version}`, the filtering block from the single-module pom, and `maven-shade-plugin` (3.6.1) bound to `package` with `<createDependencyReducedPom>false</createDependencyReducedPom>` and an `artifactSet` including only the core artifact. Build: `mvn clean package` from the root → `plugin/target/<Name>-1.0.0.jar`.

**Gradle:** `settings.gradle.kts` → `rootProject.name = "…"` and `include("core", "plugin")`. Put the shared java/release/encoding config in root `build.gradle.kts` under `subprojects { apply(plugin = "java"); … }`. `plugin/build.gradle.kts`: `compileOnly(api)`, `implementation(project(":core"))`, the `processResources` expand block, and bundle core without the shadow plugin (which needs the plugin portal and version juggling):

```kotlin
tasks.jar {
    archiveBaseName.set("@@NAME@@")
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
    dependsOn(configurations.runtimeClasspath)   // without this, Gradle 8+ fails: "implicit dependency" on core's jar task
    from({ configurations.runtimeClasspath.get().map { if (it.isDirectory) it else zipTree(it) } })
}
```

Because the API is `compileOnly`, `runtimeClasspath` holds only `core` (and `api`, if you add that module), so this bundles exactly what we want. The `dependsOn` line was found by testing: two independent test builds failed without it.

### README.md

Short, for a beginner: what the project is, the chosen versions (MC, Java, build tool), the build command, where the jar lands, how to install (copy to `plugins/`, start server, look for `enabled`), and the one-line meaning of each file. Add a "Requirements" line: JDK ≥ the Java target.

## Step 5 — Test before delivering (never skip)

Create the project, then build it and inspect the result. A project that "looks right" but was never compiled is the most common failure, so treat verification as part of creating, not an extra.

1. **Toolchain check:** `javac -version` must be ≥ the Java target (a JDK 21 cannot target 25). If it's too old, install a newer JDK if you can; otherwise say so and verify what you can.
2. **Build:** `mvn -q clean package` or `gradle build`.
3. **Inspect the jar** (find it with `find . -name '*.jar' -path '*target*' -o -name '*.jar' -path '*build/libs*'`; for multi-module pick the one that contains `plugin.yml`):
   - `unzip -p <jar> plugin.yml` — name/version/main present, `version` has no leftover `${…}`, `api-version` matches Step 3
   - main class file exists at the path `main:` points to
   - `javap -cp <jar> -v <main.class.name> | grep -E 'major version|extends|super_class'` — major version = Java target + 44 (8→52, 11→55, 16→60, 17→61, 21→65, 25→69) and the superclass is `org/bukkit/plugin/java/JavaPlugin`
   - multi-module: `unzip -l <jar>` shows the `core` classes inside
   - source contains `System.out.println("enabled")` and `System.out.println("disabled")`
4. **If dependencies can't be downloaded** (network policy, proxy 403, offline): do not retry through other routes and do not claim success. Say exactly which repository was unreachable and that the build was not verified here. Still check what you can offline (POM/Gradle syntax with `mvn -o validate`, file layout, plugin.yml). Tell the user the one command to run themselves.
5. **Optional runtime smoke test**, only if a server jar can be downloaded: put the jar in `plugins/`, start the server headless with `-Dcom.mojang.eula.agree=true` / `eula=true`, and confirm `enabled` then (after `stop`) `disabled` in the log.

Fix and rebuild until the checks pass; report anything that couldn't be checked.

## Step 6 — Deliver

Zip the project without build output (`zip -r <name>.zip <name> -x '*/target/*' '*/build/*' '*/.gradle/*'`) and send the zip. If the build succeeded, also send the built jar so they can drop it on a server immediately. Use SendUserFile when it's available; otherwise tell them where the files are.

The final message is short: the choices made (MC version, Java version and why, build tool, single/multi and the reasoning in a clause, API used), the build command, where the jar goes, and an honest line on what was verified (e.g. "built and inspected: Java 21 bytecode, extends JavaPlugin" or "dependencies could not be downloaded here, so run `mvn clean package` yourself"). Don't recap the steps.
