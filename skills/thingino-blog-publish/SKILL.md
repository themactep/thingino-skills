---
name: thingino-blog-publish
description: Write and publish articles to the thingino blog (Markdown files served by a Sinatra app, deployed via git push).
license: MIT
---
# thingino-blog-publish

Use this skill when asked to write, publish, update, unpublish or fix an
article on the thingino blog.

The blog is a minimal Sinatra app that serves `.md` files from its
`articles/` directory. Full app documentation lives in
`DOCUMENTATION.md` at the blog app root. If the local app root is
unknown, ask the user (commonly `~/www/thingino/blog`).

## When to use

- "Write a blog post about X" / "publish this on the blog".
- "Update / fix / unpublish the article about X".
- "Why doesn't my article show up?"

## Key facts

- One `.md` file = one article. No database, no admin UI, no restart —
  files take effect on the next HTTP request.
- Publishing means committing the article to the blog repo and pushing
  to the remote. The server pulls from the repo --- no scp, no manual
  file copy. The blog app root is a git working copy.

## Article rules

1. Filename: `YYYY-MM-DD-url-slug.md` (today's date, lowercase slug with
   hyphens). The slug part becomes the URL: `/url-slug`.
2. Start the file with front matter (simple `key: value` lines, no YAML
   nesting):

   ```markdown
   ---
   title: Human-readable headline
   description: One-sentence summary under 160 characters.
   author: <author name, if known>
   ---
   ```

3. Body is GFM Markdown. Start sections at `##` — never add a leading
   `# H1` when `title:` is set (it would render twice).
4. Fenced code blocks with language tags get syntax highlighting.
5. `description` is used verbatim for SEO meta and feed summaries; keep
   it under 160 characters, plain text.
6. Optional fields: `date:` (overrides filename date), `slug:`
   (overrides filename slug), `draft: true` (hides article everywhere).
7. Images are served from the `public/images/` directory (publicly
   accessible). Place article images there and reference them as
   `/images/filename.webp`. Prefer landscape `.webp` images at 600px
   wide or less. See `thingino-blog-write` for full image guidelines.

## Content style

1. Match thingino voice: technical, concise, no marketing fluff.
2. Brand is "Thingino" in prose (a proper noun); lowercase "thingino"
   is logo/wordmark styling only.
3. Prefer concrete commands, tables and short paragraphs over prose walls.
4. Link to https://thingino.com and the GitHub repo
   (https://github.com/themactep/thingino-firmware) where relevant.

## Workflow

1. Draft the article as `articles/YYYY-MM-DD-slug.md` in the local blog
   app root. For anything not explicitly approved for immediate
   publication, include `draft: true`.
2. Verify locally:
   - `cd <blog-app-root> && bundle exec rackup -p 4567 &`
   - `curl -sI http://127.0.0.1:4567/<slug>` → expect `200`
     (drafts correctly return `404`; temporarily flip the flag to preview).
   - Check rendered HTML for the title, date and body:
     `curl -s http://127.0.0.1:4567/<slug> | grep -E '<h1|<time'`
   - Stop the server afterwards (kill by port, see pitfalls).
3. Publish (after user confirms final text):
   - Ensure the article has no `draft: true` flag (remove it if present).
   - Commit and push:
     `cd <blog-app-root> && git add articles/YYYY-MM-DD-slug.md && git commit -m "..." && git push`
   - The user pulls on the server to make it live.
4. Updates: edit the same file, commit, push. Unpublish: add
   `draft: true` to the front matter, commit, push.

## Pitfalls

1. Never change the slug of a published article (breaks inbound links;
   the app has no redirects). To adjust a filename without changing the
   URL, pin the old slug with `slug:` front matter.
2. Front matter must start on line 1; both fences must be exactly `---`.
3. Slug chars are `[a-z0-9-]` only — underscores, spaces and uppercase
   in filenames are normalized, so verify the resulting URL rather than
   assuming it.
4. When killing a local test server, kill by port
   (`ss -ltnp | grep 4567`), not `pkill -f rackup` — the `-f` pattern
   can match your own shell.
5. After pushing, the article is live once the user pulls on the server.
   No further action is needed from the agent.
