---
name: release-changelog-discord-message
description: >
  Write the Discord announcement for a stable Paperclip release from the release
  changelog. Use when a release issue needs a copy-pasteable dotta-voice Discord
  post or refreshed discord_announcement document.
---

# Release Discord Announcement Skill

Write the Discord release announcement for the **stable** Paperclip release.

This is the companion to `.agents/skills/release-changelog/SKILL.md`. That skill
writes the changelog — during the beta soak it lives at
`releases/beta/v{beta-version}.md` on the `release-notes/v{beta-version}`
branch, and after the stable ships a canonicalization PR renames it to
`releases/vYYYY.MDD.P.md` (see that skill's Channel Process section). This
skill turns that file into
a single copy-pasteable Discord block, in dotta's voice, and posts it as the
`discord_announcement` document on the release issue.

## What dotta said

> This is for discord — try to follow my format. If I have a section where I
> think about the future, pull from recent issues we're working on etc.

The Discord announcement is **not** the changelog. The changelog is exhaustive;
the announcement is opinionated, in-voice, and built around the same handful of
shipped highlights plus a real "what's next" + "what's on my mind" pulled from
current Paperclip work — not invented.

## When to use

- After `release-changelog` has produced the changelog (beta-keyed on the
  `release-notes/v{beta-version}` branch during the soak, or the
  canonicalized `releases/vYYYY.MDD.P.md` after the stable ships).
- When the release issue (the one assigned by the release routine) asks for a
  Discord announcement, or has a `discord_announcement` document that needs to
  be refreshed for a new date/version.
- Never run this in isolation. The version, date, contributor list, and
  highlight set MUST match the matching changelog file — if the changelog has
  been updated, refresh this too.

## Output

A single fenced markdown code block, ready to paste into Discord. Attached as
issue document key `discord_announcement` on the release issue, and pasted
verbatim into a comment on that issue so the human can copy it out.
When Cases are enabled, also upsert the social child case described below.

```bash
PUT /api/issues/{releaseIssueId}/documents/discord_announcement
{
  "title": "Discord announcement",
  "format": "markdown",
  "body": "<the announcement>",
  "baseRevisionId": "<latest if updating>"
}
```

If the document already exists, fetch it first and pass the current
`baseRevisionId`. Never overwrite silently — if the version has changed since
the document was last written, mention what changed in the issue comment.

## Format (follow this template)

Use Discord emoji shortcodes (`:paperclip:`, `:lock:`, `:brain:` …) — NOT the
Unicode emoji. Discord renders the shortcodes; the changelog file uses prose.

```
:paperclip: :paperclip: :paperclip: CLIPPERS!!! v{VERSION} IS OUT :paperclip: :paperclip: :paperclip:

OFFICIAL TWITTER: https://x.com/papercliping - follow it, report any others

## Highlights

:emoji: **Feature Name** - one-sentence description in dotta's voice.
:emoji: **Feature Name** - …
:emoji: **Feature Name** - …

... and a long tail of {flavor of the rest}. Read the [full release notes](<github link>).

## WHATS NEXT (:motorway:  Roadmap)

* **Theme A** - one-line forward-looking blurb
* **Theme B** - …
* **Theme C** - …

## What's on my mind

* **Topic** - what's bugging dotta / what's queued / open questions
* **Topic** - …

## PRESS                              (optional — only if there is real press)

* **Outlet / Person** - what happened ([link](<x.com link>))

## WHAT I NEED FROM YOU               (optional — only if there's a real ask)

FOLLOW THE TWITTER: https://x.com/papercliping - that's the only official one
TELL ME if you're using Paperclip in your business - I want to meet you

## Community

Thank you to everyone who contributed to this release!

```
@username1, @username2, @username3
```

## In Summary

PAPERCLIP IS THE AI ORCHESTRATOR FOR HUMANS TO ACCOMPLISH 100x MORE WORK

Every single person will be managing a team of a dozen, or a hundred, or a
thousand agents and Paperclip will be the default tool to manage it all.

ITS TIME TO CLIP :paperclip: :paperclip: :paperclip:

FULL RELEASE NOTES

https://github.com/paperclipai/paperclip/blob/master/releases/v{VERSION}.md

||@everyone||
```

Notes on the template:

- The opening and closing `:paperclip: :paperclip: :paperclip:` bookends are
  part of the brand — keep them.
- Name the install channels somewhere in the post: `npx paperclipai@latest`
  for the stable, `@beta` / `@nightly` / `@canary` for earlier access, and
  Docker `:latest` moving **only** on stable releases.
- The FULL RELEASE NOTES link points at `releases/v{VERSION}.md` on
  `master` — that file exists only after the post-stable canonicalization
  PR merges. Merge it before the announcement is posted.
- Sections may be UPPERCASE or Title Case — dotta has used both. Pick a style
  and stay consistent within a single post.
- Use `||@everyone||` (Discord spoiler-wrapped) at the very end so it pings
  exactly once when the spoiler is removed by the poster.

## Language tips

These are extracted from how dotta has written the last several announcements.
Mimic this register; do not invent a "professional" tone.

- **First person, conversational.** "I want to meet companies using Paperclip",
  "what's on my mind", "if that's you let me know". Not "Paperclip is excited
  to announce".
- **ALL CAPS for excitement and asks**, especially in the opener, the section
  headers, the "WHAT I NEED FROM YOU" section, and the closing tagline. Do not
  ALL-CAPS feature descriptions.
- **One emoji shortcode per highlight bullet**, picked to evoke the feature
  (`:lock:` for secrets, `:brain:` for planning, `:mag:` for search,
  `:cloud:` for cloud / sandbox, `:jigsaw:` for plugins, `:rewind:` for
  history/restore, `:thread:` for threads, etc.).
- **Highlight bullets are one sentence**, opinionated, told from the user's
  perspective — "the cloud-secrets prereq is real now", not "added support
  for…".
- **Tail line after highlights** wraps the rest in a single sentence and links
  to the full release notes ("… and a long tail of {flavor}. Read the [full
  release notes](url).").
- **"WHATS NEXT" is forward-looking themes**, not a literal sprint list. 3–5
  bullets is the right size. Pull these from active goals, in-flight projects,
  and recent issues the team is working on — do not invent themes.
- **Highlights follow the changelog skill's delta rule**: a feature the
  previous announcement already introduced appears only for what changed
  this release, and does not headline twice for follow-through work.
- **"What's on my mind"** is dotta's personal/strategic thinking — docs gaps,
  philosophical positioning ("we're the human control plane for ai labor"),
  invitations ("if you've ever wanted to write about how you use Paperclip,
  hit me up"). Pull real tensions from recent issues/comments; do not invent.
- **Press section** is optional. Only include it if there is real press in the
  release window (a tweet, a podcast, a talk, a star milestone). No press →
  drop the section entirely.
- **"WHAT I NEED FROM YOU"** is optional. Use it for a single concrete ask
  (follow the twitter, intros, beta sign-ups). No real ask → drop it.
- **Community** is the same contributors list that's in the changelog file,
  fenced in a triple-backtick block, comma-separated `@username, @username`.
  Exclude bots and the specific folks on the changelog skill's canonical
  exclusion list — same rules.
- **The "In Summary" mission line** evolves slowly. Use the most recent
  variant unless dotta tells you otherwise. Recent variants:
  - "PAPERCLIP IS THE AI ORCHESTRATOR FOR HUMANS TO ACCOMPLISH 100x MORE WORK"
  - "PAPERCLIP WILL BE THE DEFAULT AGENT-MANAGEMENT TOOL FOR EVERY COMPANY"
  - "Paperclip will be _the_ control plane for AI agents in **every** company."
- **Closing tagline** is always `ITS TIME TO CLIP :paperclip: :paperclip:
  :paperclip:`. Keep it.

## Workflow

1. Read the matching changelog produced by `release-changelog` — the
   beta-keyed file during the soak, `releases/vYYYY.MDD.P.md` once
   canonicalized. Use the version and contributor list from that file —
   never re-derive them.
2. Resolve the parent `release` case with key `paperclip-release:vYYYY.MDD.P`.
   If it does not exist and Cases are enabled, create it using the schema in
   `.agents/skills/release-changelog/SKILL.md` before creating child cases.
3. Read the **release issue thread** (the one assigned to you that ran the
   release routine) — comments + linked issues + recent issues in the company
   are the source for `WHATS NEXT` and `What's on my mind`. Commits already
   on `origin/master` **after** the beta source commit are prime "what's
   next" material: they are literally the next release's content. Pull real
   themes, not invented ones.
4. When drafting new copy (not just a version/date refresh), read the three
   verbatim examples in [references/examples.md](references/examples.md) —
   they're the canonical voice.
5. Draft the announcement using the template above.
6. PUT it as the `discord_announcement` document on the release issue (see
   "Output" above). If updating, send the latest `baseRevisionId`.
7. Upsert the `tweet_storm` child case with `parentCaseId` set to the release
   case id, then PUT its `body` document to the announcement body.
8. Post a comment on the release issue that includes the announcement inside a
   single fenced markdown code block, so dotta can copy-paste it into Discord
   without opening the document.

## Tweet Storm Case Schema

Use this child case for the Discord/social announcement thread. The key must be
stable so retries update the same child case:

```http
POST /api/companies/:companyId/cases
{
  "caseType": "tweet_storm",
  "key": "paperclip-release:vYYYY.MDD.P:tweet-storm",
  "title": "Paperclip vYYYY.MDD.P tweet storm",
  "summary": "Social announcement thread for Paperclip vYYYY.MDD.P.",
  "status": "in_review",
  "parentCaseId": "<release-case-id>",
  "fields": {
    "schema_version": 1,
    "version": "vYYYY.MDD.P",
    "channel": "x",
    "discord_source": true,
    "post_count": 1,
    "target_audience": ["operators", "contributors", "agent-company builders"],
    "links": {
      "release_notes": "https://github.com/paperclipai/paperclip/blob/master/releases/vYYYY.MDD.P.md",
      "official_account": "https://x.com/papercliping"
    },
    "review": {
      "needs_human_copy_paste": true,
      "approved_by": null
    }
  }
}
```

Then write the body document:

```http
PUT /api/cases/:tweetStormCaseId/documents/body
{
  "title": "Paperclip vYYYY.MDD.P tweet storm body",
  "format": "markdown",
  "body": "<announcement body>",
  "changeSummary": "Draft social announcement"
}
```

If updating an existing document, fetch the case and pass the latest
`baseRevisionId`.

Do not publish to Discord. This skill only prepares the artifact.

## Review checklist

Before handing off:

1. Version + date match the matching `releases/vYYYY.MDD.P.md` exactly.
2. Contributor list matches the changelog (same exclusions: bots and the
   changelog skill's canonical excluded-folks list).
3. Highlights are a subset of the changelog Highlights — same shipped features,
   not invented or pre-alpha work.
4. `WHATS NEXT` and `What's on my mind` are pulled from real recent issues /
   active goals — not invented themes.
5. Section style (UPPERCASE vs Title Case) is internally consistent.
6. Closing tagline is `ITS TIME TO CLIP :paperclip: :paperclip: :paperclip:`
   and `||@everyone||` is the very last line.
7. Document `discord_announcement` is updated on the release issue, and the
   announcement is also posted in a comment inside a fenced code block.

This skill never posts to Discord. It only prepares the announcement artifact.
