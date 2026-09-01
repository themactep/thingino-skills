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
   Never "simply", "just", "obviously", or "of course" for steps that
   aren't simple. Cut "actually" and "in fact" — they are throat-clearing,
   not emphasis.
5. **Second person, active voice.** "You build", not "the firmware is
   built". The reader is doing this, not watching it.
6. **Let the data speak.** If a number or a log line already makes the
   point, don't add a sentence explaining what it means. Bad:
   "This confirmed the format match." The previous sentence already
   confirmed it. Bad: "This is a real, substantial difference, not
   noise." A 24% diff is already substantial — the reader can see that.
7. **No self-references.** Never write "as stated above", "more on that
   below", "from earlier in this article", or "that is the win described
   earlier". If information is needed here, put it here. If it was already
   covered, the reader remembers. Trust them.
8. **Strong sentence spines.** Avoid "X is what let Y happen" or
   "Fixing X is what made Y possible." Prefer the active verb directly:
   "With X fixed, Y ran" instead of "Fixing X is what let Y run."
9. **Explain why, not just what.** Narrative carries reasoning: where the
   fix came from, why it was needed, why structural rather than another
   patch. If a section reads like a briefing of facts ("The fix collapsed
   five writers into one gate"), its why is missing. Flow like a river,
   not a wartime report.
10. **No telegraphic fragments.** Prose is not a column. "First check: the
    clock." becomes "The first thing to check was the clock." Stacked short
    declaratives ("`ntpd` held the clock. DHCP pushed a timezone. ONVIF
    pushed the clock.") become one sentence with clauses.
11. **One metaphor, held for the whole piece.** If the fix is a gate, it
    stays a gate — not a door a paragraph later. Mixed metaphors read as
    sloppy.
12. **A header never splits a beat.** When a section builds to a question
    and its one-line answer, the header goes after the answer, not between
    the two. A header between a setup and its punch line kills the punch
    line.
13. **No worn-out phrases.** "Smoking gun" and other AI favorites are dead.
    Reach for the specific: "The clue was in the log."

## Audience

Assume a developer comfortable with Linux and git but new to embedded
firmware and Buildroot. Introduce each term of jargon once, briefly, at first
use — then use it freely. Never assume they own hardware unless the article
is explicitly hands-on.

## Structure

1. **Open with the problem, not the topic.** 1–3 short paragraphs of
   motivation the reader recognizes ("your tree becomes an archaeological
   dig"), then get to work. No throat-clearing, no "In this article we
   will...". If the reader doesn't know what's broken until paragraph 5,
   your opening is four paragraphs too long. State the problem first,
   then give the backstory.
2. Start sections at `##` (the `title:` front matter is the H1).
3. Short paragraphs (≤ 4 sentences). Prose walls are a bug.
4. Fenced code blocks with language tags for anything typed or displayed;
   comment the *why* inline when a command isn't self-evident.
5. Tables for comparisons, references, and file inventories.
6. Backticks for every path, command, variable, and config symbol in prose.
7. **ASCII punctuation only.** Normal people cannot type typographic
   characters on a keyboard, so source files stick to plain ASCII — the
   full set, not just dashes: no literal em/en dashes, curly quotes,
   arrows, middots, or box-drawing characters. Write `---` and `--` in
   prose — the renderer (kramdown) converts them to proper dashes; same
   for `...` (becomes an ellipsis). Arrows are `->` and `<-`; separators
   are `/`; tree diagrams use `|--` and `` `-- `` (`tree --charset=ascii`
   style). Two places where no conversion happens: code blocks (use a
   plain `-`) and front matter `description:` (plain hyphen, it ships
   verbatim into meta tags). Before submitting, verify with:
   `grep -Pn '[^\x00-\x7F]' articles/your-file.md` — any match is a
   non-ASCII character that must be replaced.
8. **Go easy on emphasis.** Avoid italics — they render poorly in the blog
   font and make the page look busy; let sentence structure carry the
   stress. Use **bold** sparingly: term introductions, warnings, and the
   lead-in of list items.
9. **Images.** Prefer landscape (horizontal) orientation — portrait images
   waste reading width and create awkward scrolling. Resize large images so
   the width fits within a 600px limit; anything wider breaks the blog
   layout on standard screens. Use `.webp` format for photos and
   screenshots (lossy at ~80 % quality is a good default). When converting,
   strip metadata (`-strip` in ImageMagick) for smaller files.
10. **Close with a recap.** End substantial articles with a "What you
   learned" bullet list. Five bullets max. Each bullet must be something
   the reader could apply to their own project — not a diary entry
   specific to this one investigation. Bad: "Do not trust a disassembler's
   resolved addresses on unusual MIPS relocation types." Good: "When a
   checksum algorithm is undocumented, the running binary is the spec."
   In a series, follow the recap with a one-line teaser for the next
   part.

## Series conventions

- Number the parts; keep each part readable in one sitting.
- Every part opens with `*Part N of [Series Name](...)* / [<- Part N-1](...)`.
  Part 1 opens with `*Part 1 of [Series Name](...)*` (no back-link).
- Every part ends with recap + "Next up" link.
- Escalate difficulty across parts; each part should leave the reader with a
  working result, not homework.
- When a follow-up article links back to an earlier part, verify the URL
  against the actual published slug, not the filename. Slugs can change when
  an article is renamed.

## Front matter

- Do not wrap the `title:` value in quotes. The front matter parser handles
  it as plain text. `title: My Article`, not `title: "My Article"`.
- `description:` is plain text under 160 characters. No Markdown, no HTML,
  no quotes. It ships verbatim into `<meta>` tags and feed summaries.
- `author:` is required. It is the byline on the article. Use the author's
  real name where known (`Paul Philippov`, `CaptainRon`), or `Thingino team`
  for unattributed posts. A handle is fine when that is how the author is
  known and wishes to stay pseudonymous.
- `draft: true` hides the article from the index, feeds, sitemap, and direct
  URL. Use it for anything not approved for immediate publication.

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
2. Opening paragraph states a problem the reader has — not background, not
   context, not a table of contents. The reader should know what's broken
   within the first three paragraphs.
3. No H1 in the body; sections start at `##`.
4. `description` front matter is a plain-text sentence under 160 characters.
   No quotes, no Markdown.
5. `author:` front matter present, with a real name, a pseudonymous handle,
   or `Thingino team`.
6. Brand and terminology table respected.
7. Humor count per section ≤ 1; zero jokes inside technical content.
8. Recap present with ≤ 5 generalizable bullets. Series links intact in
   both directions, verified against published slugs.
9. No self-references ("as stated above", "more on that below", "from
   earlier in this article").
10. No filler words: "simply", "just", "obviously", "actually", "in fact",
    "of course".
11. No editorializing confirmation sentences ("This confirmed the match",
    "This is a real difference"). If the data already made the point,
    trust it.
12. ASCII only. Run `grep -Pn '[^\x00-\x7F]' articles/your-file.md`.
    Any match is a non-ASCII character — replace it.
13. `title:` front matter is not wrapped in quotes.
14. Read it aloud once — anything you stumble on, the reader will too.
15. Every major turn explains why, not just what — the reasoning is on the
    page, not implied.
16. No telegraphic fragments or stacked "X did Y" columns.
17. One metaphor held consistently throughout (no gate/door drift).
18. No header splits a question from its answer.
19. No worn-out AI phrases ("smoking gun" and friends).
20. No repeated construction — a phrase used three times is a tic; vary it.

## Notes

- Reference article for tone and format: the blog's welcome post
  (`articles/2025-01-15-welcome-to-the-thingino-blog.md`). Note it predates
  the branding rule above — don't imitate its lowercase "thingino" in prose.
- Reference series for long-form structure: "The Thingino Developer's
  Journey" (parts 1–6, from first build to git worktrees).
- Mechanics (filename, front matter, preview, scp deploy, slug pitfalls):
  see `thingino-blog-publish`.
