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
| A topic or description ("I want a feed about jazz") | **Describe** | If the angle is clear, propose a first-pass menu of lanes/terms/exclusions for that topic. If the topic is broad/ambiguous, ask one intent question first (see Interview Process). |
| A list of terms or keywords | **Refine** | Structure the list into proper syntax, then propose the specific additions and exclusions you'd make for that topic. |
| Something vague or just "create feed rules" | **Open** | Ask: "What kind of Bluesky feed do you want to create? Describe the topic, paste a rough list of terms, or just tell me the vibe." |

## Interview Process

Lead with what you know. You already understand most topics — their sub-areas, key terms, and the noise that pollutes them. Don't ask the user to supply that knowledge; **propose it and let them react.** Ask questions one at a time, prefer multiple choice, and only ask a question if its answer would change the output.

**The flow:**

1. **Fix the angle (one intent question, broad topics only).** If the topic is narrow (e.g. "posts about Fugazi"), the angle is already clear — skip to step 2. If it's broad or ambiguous (e.g. "music"), ask exactly one question to pin the heart of the feed. Example: *"What's the heart of this feed — a genre/scene, an activity (gigs, gear, releases), or a vibe?"*

2. **Propose a specific menu.** Lead with the actual lanes, terms, and noise-to-cut you'd use for this topic, and ask the user to confirm or narrow. Don't ask "what subgenres?" — name them. Example for jazz:

   > "For a jazz feed I'd pull in **bebop, hard bop, modal, fusion, free jazz, vocal jazz**, plus reviews and interviews. I'd cut smooth-jazz / elevator-music noise and gear-for-sale spam. Want all those lanes, or focus a few?"

3. **Generate a first pass, then offer an exit.** After the menu is confirmed (or immediately, for narrow topics), generate. Always close with: "Here's a first pass. Want me to refine it, or is this ready to paste?"

**What to cover — propose, don't interrogate.** Each row below is something you *offer*, scaled to topic complexity. A narrow topic touches 2-3 rows; a broad one touches more.

| Phase | What you do | When |
|-------|-------------|------|
| **Intent** | Ask the single angle question above. | Broad/ambiguous topics only |
| **Scope & exclusions** | *Name* the noise you expect for this specific topic ("for a film feed, I'd cut showtimes/ticket spam and streaming-deal posts") and ask the user to confirm or add. | Always — exclusions are half the value of a good feed |
| **Media & safety** | Suggest `!images-only` / `!videos-only` for inherently visual topics, and `!no-adult` for feeds that should stay SFW — as a proposal, not an open question. | Visual/format topics, or any feed that should stay SFW |
| **Naming** | Suggest a concrete name; don't just ask for one. | Always |
| **Tuning** | Propose a slant (discussion vs. news vs. art) if the topic is broad, and name 1-2 well-known accounts if you're confident they're real (see confidence rules under After Generating). | Broad or ambiguous topics |
| **Structure** | If the user's intent implies intersection ("X *that are also* Y"), propose splitting into groups (see Grouping). | Only when intent implies intersection |

Do not ask standalone questions that don't change the output (e.g. a generic "who is this feed for?" with no follow-on effect). If audience matters, fold it into the menu ("aimed at newcomers, so I'd keep terms broad — sound right?").

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

# Keywords — core indie-rock terms + key press
"indie rock"
"indie pop"
shoegaze
dreampop
postpunk
"post punk"
^Pitchfork
"album review"

# Hashtags — likely tags (some may be quiet)
#indierock
#shoegaze
#dreampop
#newmusic

# Exclusions — resale / merch noise
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

# Exclusions (global) — resale + spam
-"concert tickets"
-"for sale"
-@spambot.bsky.social

# Settings
!no-replies
!lang:en
```

Keep section-header comments short and intent-naming (e.g. `# Exclusions — resale noise`). The fuller "what I included and why" belongs in chat, not in the pasteable block — keep the paste clean.

## After Generating

### How confident to be

Your knowledge is reliable for some suggestion types and not others. Calibrate:

- **Keywords, phrases, subgenres, key terms** — suggest freely and specifically. This is your strong suit; be generous and concrete (name the actual sub-areas and signal words for the topic).
- **`@handles`** — only suggest clearly-real, well-known public figures or organizations, and **always flag them**: "double-check this handle exists on Bluesky." Never invent plausible-looking handles. When unsure, leave them out.
- **`#hashtags`** — suggest them, but note they may not all be active on Bluesky ("these are likely tags — some may be quiet").

Scan the output for obvious gaps before presenting it. Proactively suggest improvements, and make them specific to *this* topic rather than generic:
- If no hashtags are included, suggest the likely tags for the topic ("Bluesky users tag these #jazz / #jazzmusic — want them in?"), noting some may be quiet.
- If exclusions feel thin, *name* the noise pattern unique to this subject ("art feeds get flooded with print-shop/Redbubble promos — want those cut?") rather than saying "every topic has noise."
- If the topic is broad, propose a specific way to tighten it (a sub-lane to focus on), not just "want to narrow?".
- If the topic is inherently visual (photography, art, film, memes), suggest `!images-only` or `!videos-only` — but only when the user wants media-only; these hard-narrow the feed.
- If the feed should stay family-friendly or workplace-safe, suggest `!no-adult`.

After presenting the block, give a one-or-two-line "here's what I included and why" summary in chat, then offer **concrete, topic-specific next moves** — instantiate the options for this feed rather than listing them abstractly. For a jazz feed that might be:
- "Add the fusion / jazz-funk lane?"
- "Tighten to discussion-only (reviews, interviews, threads)?"
- "Add a `!no-replies` setting to cut reply chatter?"
- "Looks right — ready to paste?"

The underlying moves are always: add/remove terms, tighten (more exclusions / narrower scope) or broaden, add groups for intersection logic, or finalize. Phrase them as specific suggestions for the current topic, not a generic menu.

## Quality Checklist

Before presenting the final output, verify:
1. Every rule uses the correct prefix for its type
2. The output is a valid fenced code block
3. There are at least 2-3 exclusion rules (every topic has noise)
4. The keyword count is 10-20 — quality over quantity
5. Specific terms are preferred over generic ones
6. Any suggested `@handles` are clearly-real public figures/orgs AND carry a "verify this handle exists on Bluesky" caveat — no fabricated or guessed handles
7. The feed has a name in the first comment line
8. Suggestions (terms, exclusions, hashtags) are specific to this topic — no generic filler like "every topic has noise"

## Constraints

- Do NOT call any API or write any files — output is text only
- Do NOT validate handles against Bluesky (future enhancement)
- Do NOT fetch or inspect existing feeds (future enhancement)
