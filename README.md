# hashnode-blog

Markdown source for my [Hashnode](https://hashnode.com) blog posts, published through Hashnode's **Publish from GitHub** integration.

Author: Jaydeep Kanzariya

Blog: https://jaydeepkanzariya.hashnode.dev (use `domain: jaydeepkanzariya.hashnode.dev` in frontmatter)

## How publishing works

- The Hashnode GitHub app is installed on this repo only.
- Each `.md` file in the repo **root** is one post. Hashnode reads the frontmatter at the top of the file.
- Pushing to `main` creates or updates the post. Check **GitHub Integration → View logs** in the Hashnode dashboard if a post doesn't appear.

## Rules I follow

1. One post per file, in the repo root. Hashnode's docs say posts must not be in a folder.
2. File name = slug, e.g. `linux-file-system-hunting.md`.
3. **Never change a slug after publishing.** It is the update key, and changing it can create a duplicate post.
4. Required frontmatter: `title`, `slug`, `tags` (max 5), `domain`.
5. `cover` must be a Hashnode CDN URL (upload the image at the Hashnode Uploader first), e.g. `https://cdn.hashnode.com/uploads/covers/<id>/<file>.png`. Local copies live in `covers/`.
6. First push of a new post uses `saveAsDraft: true`. Remove it to publish.
7. Max 10 file changes per commit.

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

## Adding a new post

1. Create `<slug>.md` in the root with the frontmatter from an existing post.
2. Upload the cover at the Hashnode Uploader, paste the CDN URL into `cover`, and keep a copy in `covers/`.
3. Commit and push. Review the draft in the Hashnode dashboard, then remove `saveAsDraft` and push again.
