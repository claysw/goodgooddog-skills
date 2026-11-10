# goodgooddog skills

Claude Code skills from [goodgooddog](https://goodgooddog.com): build and tune custom Bluesky feeds, and turn app walkthrough recordings into FigJam boards.

This repository is a Claude Code **plugin marketplace**. It ships two plugins:

| Plugin | Skill | What it does |
|--------|-------|--------------|
| `feed-creator` | `generate-feed-rules` | Interviews you about a feed idea, then outputs **shorthand** you paste into goodgooddog's bulk import to create the feed. |
| `commentary-to-figjam` | `commentary-to-figjam` | Turns a screen recording you talked over into a FigJam board with a screenshot and ticket-ready note per key moment, for any app. See [its README](plugins/commentary-to-figjam/README.md). |

## Install

```
/plugin marketplace add claysw/goodgooddog-skills
/plugin install feed-creator@goodgooddog
/plugin install commentary-to-figjam@goodgooddog
```

Then invoke it in Claude Code:

```
/feed-creator:generate-feed-rules
```

…or just describe the feed you want ("I want a feed about jazz") and the skill will pick it up.

## What you get

The skill runs a short conversational interview — topic, who it's for, what to exclude, whether to require images/video or keep it SFW — and produces a block of shorthand like:

```
# Jazz Discussion Feed

jazz
bebop
"hard bop"
#jazzmusic

# Exclusions
-"concert tickets"
-#nft

# Settings
!no-replies
!lang:en
```

Paste that into goodgooddog's **Import Feed from Shorthand** tool and the feed is created.

See the [shorthand syntax reference](plugins/feed-creator/skills/generate-feed-rules/SKILL.md#shorthand-syntax-reference) for the full set of prefixes and flags.

## Updating

Installed users receive new versions when the plugin's version is bumped. To update:

```
/plugin update feed-creator@goodgooddog
```

## Releasing a new version (maintainers)

Updates are delivered **only when the version changes** — pushing commits alone is not enough.

1. Edit the plugin's `SKILL.md` under `plugins/<plugin>/skills/`.
2. Bump `version` in `plugins/<plugin>/.claude-plugin/plugin.json` (e.g. `1.0.0` → `1.0.1`).
3. Validate: `claude plugin validate .`
4. Commit and push.

## License

[MIT](LICENSE)
