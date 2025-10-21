# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

title-updater is a Minecraft Bukkit library that updates inventory titles dynamically with Adventure component support. Published to Maven Central as `net.infumia:title-updater`.

## Architecture

### Multi-Version NMS Support

The project uses a **version-abstraction pattern** to support multiple Minecraft versions (1.8.8 through 1.21.6):

- **common module**: Public API and version detection logic
  - `TitleUpdater.update()` - Main entry point
  - `NmsProvider` - Dynamically loads version-specific implementations
  - Uses pattern `net.infumia.titleupdater.versions.NmsV{major}_{minor}[_{patch}]`

- **nms/common**: Shared interface `Nms` with `updateTitle()` method

- **nms/v{version}**: Version-specific implementations (e.g., `nms/v1.8.8`, `nms/v1.21.6`)
  - Each implements `Nms` interface using version-specific NMS classes
  - Singleton pattern via `public static final Nms INSTANCE`

### Build System Architecture

Custom Gradle conventions in `buildSrc/src/main/kotlin/net/infumia/gradle/`:

- **nms.kt**: `applyNms()` - Configures ShadowJar to merge all NMS module JARs into one
  - Searches for `-reobj.jar` first (obfuscation-mapped), falls back to regular JAR
  - Excludes annotation packages to reduce size

- **publish.kt**: `applyPublish()` - Maven Central publishing setup
- **spotless.kt**: Code formatting (Prettier for Java, ktfmt for Kotlin)

Version discovery algorithm (NmsProvider):
1. Parses server version from Bukkit
2. Iterates backwards from current patch/minor version
3. Finds first matching NMS implementation class
4. Throws if no compatible version found

## Build Commands

```bash
# Build all modules (includes shadowJar)
./gradlew build

# Build specific module
./gradlew :common:build
./gradlew :nms-v1_20_6:build

# Run code formatter
./gradlew spotlessApply

# Check formatting without applying
./gradlew spotlessCheck

# Publish to Maven Local
./gradlew publishToMavenLocal

# Publish to Maven Central (requires credentials and -Psign-required)
./gradlew publish -Psign-required
```

## Development Workflow

### Adding a New Minecraft Version

1. Add version to `settings.gradle.kts` `registerNmsModules()` list
2. Create `nms/v{version}/build.gradle.kts`:
   ```kotlin
   import net.infumia.gradle.applyJava

   applyJava()

   dependencies {
       compileOnly(project(":nms-common"))
       compileOnly(libs.minecraft.{version}.nms) { isTransitive = false }
   }
   ```
3. Implement `NmsV{major}_{minor}[_{patch}]` class in appropriate package
4. Update `gradle/libs.versions.toml` with new Minecraft dependency

### Testing Changes

No automated tests exist. Manual testing requires:
1. Build with `./gradlew build`
2. Copy shadowed JAR from `common/build/libs/` to test server
3. Test with different inventory types (avoid WORKBENCH, ANVIL, CRAFTING, PLAYER, CREATIVE)

## Key Technical Details

- **Component Support**: Detects Adventure API via `Internal.supportsComponents()` for 1.16+ Paper servers
- **Inventory Type Filtering**: Certain inventory types cannot have titles updated (returns early)
- **Title Format**: Accepts String, Kyori Component, or Gson JsonElement
- **Packet-based**: Uses `PacketPlayOutOpenWindow` to update titles without closing inventory
