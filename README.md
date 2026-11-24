# Zyphryon Manual

Usage manuals for the Zyphryon libraries, packaged as [Claude Code](https://code.claude.com) plugins. Each plugin
holds one skill: a tutorial-length guide an agent reads on demand while building a program or game with the library.

| Plugin | Library | Covers |
|---|---|---|
| `zyphryon-engine` | [Engine](https://github.com/Zyphryon/Engine) | getting started, base, math, scene (ECS), content, runtime, graphics, audio, AI, networking and replication |
| `zyphryon-ui` | [UI](https://github.com/Zyphryon/UI) | hosting the UI, themes, layouts, widgets, data binding, text and images |
| `zyphryon-studio` | [Studio](https://github.com/Zyphryon/Studio) | the asset baker: textures, trackers, models, materials, fonts, sounds, output formats |
| `zyphryon-database` | [Database](https://github.com/Zyphryon/Database) | connections, queries, SQLite, PostgreSQL, digests |

## Install

In Claude Code:

```
/plugin marketplace add Zyphryon/Manual
/plugin install zyphryon-engine@zyphryon
/plugin install zyphryon-ui@zyphryon
/plugin install zyphryon-studio@zyphryon
/plugin install zyphryon-database@zyphryon
```

Install only the ones you use. A skill loads when a task matches its description; you can also call it directly,
for example `/zyphryon-engine:zyphryon-engine`.

## Layout

```
.claude-plugin/marketplace.json          the "zyphryon" marketplace
<plugin>/.claude-plugin/plugin.json      one per library
<plugin>/skills/<name>/SKILL.md          overview, chapter index, common traps
<plugin>/skills/<name>/references/*.md   one chapter per area
```

## Versions

The manuals describe the libraries' current main branches. When a chapter and the code disagree, the code is right;
please open an issue.
