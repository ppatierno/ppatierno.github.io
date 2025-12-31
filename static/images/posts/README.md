# Blog Post Images

This directory is for **featured images** (hero images) that appear in blog post listings and at the top of individual blog posts.

## Usage

1. Place your featured images in this directory: `static/images/posts/`
2. Reference them in your blog post front matter:

```yaml
---
title: "My Blog Post"
date: 2025-01-15
image: "images/posts/my-featured-image.jpg"
---
```

## Image Paths

- Files in `static/images/posts/` are served at `/images/posts/`
- In front matter, use: `image: "images/posts/filename.jpg"`
- The templates will automatically prepend `/` to make it an absolute path

## Inline Images in Content

For images that appear **within the blog post content** (not the featured image), you have two options:

1. **Per-post assets directory** (recommended for post-specific images):
   - Create `content/posts/your-post-slug/assets/` directory
   - Reference them as: `![Alt text](./assets/image-name.png)`
   - These are bundled with the post and work well for diagrams, screenshots, etc.

2. **Global static directory**:
   - Place in `static/images/posts/`
   - Reference as: `![Alt text](/images/posts/image-name.png)`

## Example Structure

```
static/
  images/
    posts/
      my-post-featured.jpg      # Featured image (in front matter)
      shared-diagram.png        # Can be used inline too

content/
  posts/
    my-post/
      index.md                  # Blog post content
      assets/                   # Post-specific assets
        diagram1.png            # Referenced as ./assets/diagram1.png
        screenshot.png          # Referenced as ./assets/screenshot.png
```
