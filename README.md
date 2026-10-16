# minecraft-plugin (Claude skill)

A Claude skill that creates **ready-to-build Minecraft server plugins** in Java.

It asks you for:

1. the Minecraft version (1.8 up to the latest, including the 2026 `26.x` versions)
2. the Java version (only versions valid for your Minecraft version; lowest by default)
3. the build tool (Maven or Gradle)

Then it decides between a single-module and multi-module project, scaffolds it
(main class extends `JavaPlugin`, prints `enabled` / `disabled`, with getting-started comments),
builds it and checks the jar before handing it over.

## Install

**As a Claude Code plugin**:

```
/plugin marketplace add Parsa3323/minecraft-plugin-skill
/plugin install minecraft-plugin@minecraft-plugin-marketplace
```

**As a plain skill:** copy the folder `skills/minecraft-plugin/` into
`~/.claude/skills/` (all projects) or `.claude/skills/` (one project).

**On claude.ai:** upload `skills/minecraft-plugin/SKILL.md` through Settings, Skills.

## Use

Just ask: *"make me a Minecraft plugin called MyPlugin"*.

## Notes

- Java requirements per Minecraft version and API coordinates were checked in October 2026.
  Re-check them when a new Minecraft version comes out.
- The skill was tested on Gradle builds against a stand-in API. Maven builds and real
  Spigot/Paper downloads were not tested.

## License

MIT
