# hashnode-blog

Markdown source for my [Hashnode](https://hashnode.com) blog posts. This repo is the backup and version history. Posts are published manually in the Hashnode editor (the GitHub integration needs a paid plan).

Author: Jaydeep Kanzariya

Blog: https://jaydeepkanzariya.hashnode.dev

## How publishing works

1. Write the post as `<slug>.md` in the repo root.
2. In Hashnode, create a new article and paste the markdown **below the frontmatter** (the part after the second `---`).
3. Set the title, subtitle, slug, tags and cover in the editor's settings, using the values from the frontmatter.
4. Fill in the SEO title and description in the article's settings.
5. Publish.
6. Commit any changes here so the repo matches what is live.

## Rules I follow

1. One post per file, in the repo root.
2. File name = slug, e.g. `linux-file-system-hunting.md`.
3. Never change a slug after publishing, since it changes the post URL.
4. Frontmatter is a record of the post's settings. Hashnode does not read it, so copy the values into the editor by hand.
5. Cover images are uploaded in Hashnode, e.g. `https://cdn.hashnode.com/uploads/covers/<id>/<file>.png`. Keep a local copy in `covers/`, named after the slug.
6. Images inside a post are also uploaded to Hashnode, and the CDN URL goes in the markdown. Do not link to local files.
7. Max 5 tags per post.
8. If I edit a published post in Hashnode, copy the change back here.

## Frontmatter template

```yaml
---
title: Post title
slug: post-slug
subtitle: One-line summary shown under the title
tags: tag1, tag2, tag3
domain: jaydeepkanzariya.hashnode.dev
cover: https://cdn.hashnode.com/uploads/covers/<id>/<file>.png
seoTitle: Search title (around 60 characters)
seoDescription: Search description (around 155 characters)
enableToc: true
---
```

## Adding a new post

1. Create `<slug>.md` in the root with the frontmatter template above.
2. Upload the cover at the Hashnode Uploader, paste the CDN URL into `cover`, and keep a copy in `covers/`.
3. Create a new article in the Hashnode editor, paste the body (below the frontmatter), and copy the title, subtitle, slug, tags, cover and SEO fields from the frontmatter.
4. Turn on the table of contents if `enableToc: true`.
5. Preview the article, then publish.
6. Open the live post and check the cover, headings, code blocks and links.
7. Add a row to the Posts table below, then commit and push.

## Git setup for this repo

Identity and hooks are set locally, not globally:

- `user.name` = Jaydeep Kanzariya, `user.email` = jkanazariya@gmail.com
- Git hooks are disabled (`core.hooksPath` points to a folder that doesn't exist). To enable them again: `git config --local --unset core.hooksPath`.

## Structure

```
hashnode-blog/
├── README.md
├── linux-file-system-hunting.md
└── covers/
    └── linux-file-system-hunting.png
```

## Posts

| Post | Slug |
|---|---|
| A Linux machine gave away its whole setup through its files | `linux-file-system-hunting` |
