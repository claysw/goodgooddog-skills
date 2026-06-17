---
name: generate-feed-rules
description: Generate Bluesky feed rules in goodgooddog shorthand syntax. Use this skill whenever a user wants to create, build, or refine feed rules for Bluesky — including when they say "I want a feed about...", "help me filter Bluesky posts", "curate my feed", "what keywords should I use for my feed", paste a rough keyword list, describe a topic for a feed, mention goodgooddog, or ask about feed curation, feed filtering, or feed configuration. Also triggers on "build a feed", "create feed rules", "feed for [topic]", "set up a Bluesky feed", or any request involving Bluesky content filtering by keywords, hashtags, or accounts.
---

# Bluesky Feed Rules Generator

Create feed rules for [goodgooddog](https://goodgooddog.com) through a short conversational interview. The output is shorthand text the user pastes into the app's bulk import field.

## Detecting Intent

When the skill loads, look at the user's message (or `$ARGUMENTS`, if invoked as a slash command) to determine which mode to start in:

| What the user said | Mode | First action |
|---------------------|------|--------------|
| A topic or description ("I want a feed about jazz") | **Describe** | Ask 1 scoping question, then generate a first pass |
| A list of terms or keywords | **Refine** | Structure the list into proper syntax, suggest additions and exclusions |
| Something vague or just "create feed rules" | **Open** | Ask: "What kind of Bluesky feed do you want to create? Describe the topic, paste a rough list of terms, or just tell me the vibe." |

## Interview Process

Ask questions **one at a time**. Prefer multiple choice when possible. The depth of the interview scales to the complexity of the topic:

**Narrow topic** (e.g., "posts about Fugazi") → 2-3 questions, then generate.
**Broad topic** (e.g., "music") → 5+ questions covering subgenres, tone, accounts, exclusions.

| Phase | Ask | When |
|-------|-----|------|
| **Core** | What's the topic? Who is this feed for? | Always |
| **Scope** | What should be excluded? What kind of noise do you expect? | Always — exclusions are half the value of a good feed |
| **Media & safety** | Only posts with images or video? Keep it free of adult content? | Visual/format-specific topics (photos, art, film), or any feed that should stay SFW |
| **Naming** | What should the feed be called? | Always — suggest a name if the user doesn't offer one |
| **Tuning** | Tone? Discussion vs. news vs. art? Specific accounts to include? | Only for broad or ambiguous topics |
| **Structure** | "Should X and Y be separate groups where both must match?" | Only when the user's intent implies intersection logic (see Grouping below) |

After generating a first pass, always offer an exit: "Here's a first pass. Want me to refine it, or is this ready to paste?"

## Shorthand Syntax Reference

Each line is one rule. The prefix determines the type:

| Syntax | Type | Example |
|--------|------|---------|
| `word` | keyword include | `fugazi` |
| `^Word` | keyword include (case-sensitive) | `^Fugazi` |
| `"multi word"` | keyword include (phrase) | `"hard bop"` |
| `^"Multi Word"` | keyword include (phrase, case-sensitive) | `^"Miles Davis"` |
| `-word` | keyword exclude | `-spam` |
| `-^Word` | keyword exclude (case-sensitive) | `-^SPAM` |
| `#tag` | hashtag include | `#indierock` |
| `-#tag` | hashtag exclude | `-#politics` |
| `@handle.bsky.social` | user include | `@jazzcat.bsky.social` |
| `-@handle.bsky.social` | user exclude | `-@spammer.bsky.social` |
| `!no-replies` | exclude replies | |
| `!images-only` | require images | |
| `!videos-only` | require videos | |
| `!no-adult` | exclude adult / self-labeled content | |
| `!match-links` | also match keywords against linked domains | |
| `!lang:xx` | language filter | `!lang:en,es` |
| `# text` | comment (ignored by parser) | `# Jazz feed` |

**When to use special prefixes:**
- `^` (case-sensitive): Only for proper nouns, acronyms, and band/brand names where the lowercase form is a common word (e.g., `^Fugazi` not `fugazi`, `^ADHD` not `adhd`). Phrases are case-insensitive by default; use `^"..."` only when the phrase is a proper noun (e.g., `^"Miles Davis"`).
- `"quoted phrases"`: Only when word order matters and the individual words would be too noisy alone (e.g., `"hard bop"` because `hard` alone matches too much, but `bebop` doesn't need quotes)

### Grouping

Rules can be organized into groups separated by `---`:
- **Within a group:** rules are OR'd (any match includes the post)
- **Between groups:** groups are AND'd (all groups must match)

**When to use groups:** If the user says "posts about X **that are also** Y" or "X **AND** Y" — that's intersection logic, use groups. If they just list topics or keywords, keep it flat (one group). Default to flat unless the user's intent clearly implies intersection.

## Output Format

Always output as a fenced code block with `#` comments explaining each section.

### Simple example (flat, one group)

```
# Indie Rock Feed
# Generated for goodgooddog bulk import

# Keywords
"indie rock"
"indie pop"
shoegaze
dreampop
postpunk
"post punk"
^Pitchfork
"album review"

# Hashtags
#indierock
#shoegaze
#dreampop
#newmusic

# Exclusions
-"concert tickets"
-"for sale"
-"buy now"
-#nft

# Settings
!no-replies
!lang:en
```

### Compound example (with groups)

Use this pattern when the user wants intersection logic — e.g., "jazz posts that are discussion, not just links."

```
# Jazz Discussion Feed
# Posts must be about jazz AND be discussion content

# Group 1 — jazz terms (any match)
jazz
bebop
"hard bop"
"modal jazz"
#jazz
#jazzmusic

---

# Group 2 — discussion signals (any match)
review
interview
opinion
"album of the year"
"what do you think"
thread

# Exclusions (global)
-"concert tickets"
-"for sale"
-@spambot.bsky.social

# Settings
!no-replies
!lang:en
```

## After Generating

Scan the output for obvious gaps before presenting it. Proactively suggest improvements:
- If no hashtags are included, suggest relevant ones ("Bluesky users often tag posts with #jazz — want me to add hashtag rules?")
- If exclusions feel thin, suggest common noise patterns for that topic
- If the topic is broad, ask whether the scope should be tightened
- If the topic is inherently visual (photography, art, film, memes), suggest `!images-only` or `!videos-only` — but only when the user wants media-only; these hard-narrow the feed
- If the feed should stay family-friendly or workplace-safe, suggest `!no-adult`

Then offer these refinement options:
- **Add/remove** specific terms
- **Tighten** (more exclusions, narrower scope) or **broaden** (more keywords, related hashtags)
- **Add groups** for intersection logic
- **Finalize** — confirm it's ready to paste

## Quality Checklist

Before presenting the final output, verify:
1. Every rule uses the correct prefix for its type
2. The output is a valid fenced code block
3. There are at least 2-3 exclusion rules (every topic has noise)
4. The keyword count is 10-20 — quality over quantity
5. Specific terms are preferred over generic ones
6. No `@handles` are suggested unless you're confident they exist on Bluesky (when in doubt, leave them out)
7. The feed has a name in the first comment line

## Constraints

- Do NOT call any API or write any files — output is text only
- Do NOT validate handles against Bluesky (future enhancement)
- Do NOT fetch or inspect existing feeds (future enhancement)
