---
name: thingino-blog-write
description: Write and edit thingino blog articles in the established voice, structure, and terminology so the blog stays consistent.
license: MIT
---
# thingino-blog-write

Use this skill when drafting, rewriting, or reviewing **article content** for
the thingino blog. For filenames, front matter, local preview, and deployment
mechanics, use the companion skill `thingino-blog-publish` — this skill is
about what the words say; that one is about how they ship.

## When to use

- "Write a blog post / article / series about X."
- "Edit / tighten / review this draft for the blog."
- "Does this match the blog style?"

## Voice

1. **A friendly expert peer.** Write as one developer to another: direct,
   warm, never corporate. No marketing fluff, no exclamation-point
   enthusiasm, no "delve", no "in today's fast-paced world".
2. **Technical and concrete.** Prefer a real command over a description of
   the command. Prefer numbers over adjectives ("30–60 minutes", not "a
   while"). Never invent flags, paths, or output — verify every command
   against the actual thingino-firmware tree or docs before quoting it.
3. **Humor is a spice, not a course.** At most one light aside per section,
   always in prose, never inside commands, tables, or facts. Good: "four
   screws and one betrayed warranty sticker". Bad: joke names in example
   commands.
4. **Confident, not condescending.** Demystify instead of gatekeeping:
   "it's less mystical than it looks", "this is normal, this is the hobby".
   Never "simply" or "just" for steps that aren't simple.
5. **Second person, active voice.** "You build", not "the firmware is
   built". The reader is doing this, not watching it.

## Audience

Assume a developer comfortable with Linux and git but new to embedded
firmware and Buildroot. Introduce each term of jargon once, briefly, at first
use — then use it freely. Never assume they own hardware unless the article
is explicitly hands-on.

## Structure

1. **Open with the problem, not the topic.** 1–3 short paragraphs of
   motivation the reader recognizes ("your tree becomes an archaeological
   dig"), then get to work. No throat-clearing, no "In this article we
   will...".
2. Start sections at `##` (the `title:` front matter is the H1).
3. Short paragraphs (≤ 4 sentences). Prose walls are a bug.
4. Fenced code blocks with language tags for anything typed or displayed;
   comment the *why* inline when a command isn't self-evident.
5. Tables for comparisons, references, and file inventories.
6. Backticks for every path, command, variable, and config symbol in prose.
7. **ASCII punctuation only.** Normal people cannot type typographic
   characters on a keyboard, so source files stick to plain ASCII. Never
   type literal em/en dashes (—, –): write `---` and `--` in prose — the
   renderer (kramdown) converts them to proper dashes. Same for `...`
   (becomes an ellipsis). Two places where no conversion happens: code
   blocks (use a plain `-`) and front matter `description:` (plain hyphen,
   it ships verbatim into meta tags).
8. **Go easy on emphasis.** Avoid italics — they render poorly in the blog
   font and make the page look busy; let sentence structure carry the
   stress. Use **bold** sparingly: term introductions, warnings, and the
   lead-in of list items.
9. **Close with a recap.** End substantial articles with a short
   "What you learned" bullet list. In a series, follow it with a one-line
   teaser for the next part.

## Series conventions

- Number the parts; keep each part readable in one sitting.
- Every part opens with `*Part N of [Series Name](...)* · [← Part N-1](...)`.
- Every part ends with recap + "Next up" link.
- Escalate difficulty across parts; each part should leave the reader with a
  working result, not homework.

## Terminology and branding

| Write | Not |
|---|---|
| Thingino (a proper noun — capitalize in prose and titles) | THINGINO, ThingIno |
| Buildroot, U-Boot, ONVIF, RTSP | buildroot, uboot, Onvif |
| SoC, T31X, GC2053 (chip caps as vendors print them) | soc, t31x in prose (fine inside code/paths) |
| defconfig, fragment, overlay, streamer | ad-hoc synonyms for established terms |
| camera (the config/device) | board, target (except quoting `BOARD=`) |
| `master` / `stable` branches | main |

Link [thingino.com](https://thingino.com), the
[GitHub repo](https://github.com/themactep/thingino-firmware), and in-repo
docs (`docs/<file>.md`) wherever they help; prefer linking a doc over
re-explaining it at length.

Lowercase "thingino" is the logo/wordmark styling, not a prose rule. In
written text, treat the name like any proper noun: Thingino.

## Review checklist

Before handing a draft over for publication:

1. Every command copy-paste-verified against the firmware tree (or clearly
   marked as illustrative).
2. Opening paragraph states a problem the reader has.
3. No H1 in the body; sections start at `##`.
4. `description` front matter is a plain-text sentence under 160 characters.
5. Brand and terminology table respected.
6. Humor count per section ≤ 1; zero jokes inside technical content.
7. Recap present; series links intact in both directions.
8. Read it aloud once — anything you stumble on, the reader will too.

## Notes

- Reference article for tone and format: the blog's welcome post
  (`articles/2025-01-15-welcome-to-the-thingino-blog.md`). Note it predates
  the branding rule above — don't imitate its lowercase "thingino" in prose.
- Reference series for long-form structure: "The Thingino Developer's
  Journey" (parts 1–6, from first build to git worktrees).
- Mechanics (filename, front matter, preview, scp deploy, slug pitfalls):
  see `thingino-blog-publish`.
