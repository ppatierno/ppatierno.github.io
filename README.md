# Personal Website with Hugo and Massively Theme

This is a personal website built with [Hugo](https://gohugo.io/) static site generator using the [Massively](https://html5up.net/massively) theme from HTML5 UP.

## Getting Started

### Prerequisites

- [Hugo](https://gohugo.io/installation/) installed on your system

On Fedora:
```bash
sudo dnf install hugo
```

### Development

To start the development server:

```bash
hugo server
```

The site will be available at `http://localhost:1313`

### Building

To build the site for production:

```bash
hugo
```

The generated static files will be in the `public/` directory.

## Project Structure

```
.
├── archetypes/      # Content templates
├── content/         # Content files (markdown)
│   ├── about-me.md  # About page
│   └── blog/        # Blog posts
├── static/          # Static files (images, etc.)
│   └── images/      # Image files
├── themes/          # Hugo themes
│   └── massively/   # Massively theme
│       ├── layouts/ # Template files
│       └── static/  # Theme assets (CSS, fonts)
├── .github/
│   └── workflows/
│       └── hugo.yml # GitHub Actions deployment
├── hugo.toml        # Hugo configuration
└── public/          # Generated static site (after build)
```

## Configuration

Edit `hugo.toml` to customize:
- Site title and metadata
- Theme parameters (intro, social links)
- Menu items

## Content

### About Page
Edit `content/about-me.md` to update your about page.

### Blog Posts
Create new blog posts:
```bash
hugo new blog/my-post-title.md
```

## GitHub Pages Deployment

This site automatically deploys to GitHub Pages using GitHub Actions:

1. Go to repository **Settings → Pages**
2. Under "Build and deployment", set **Source** to **GitHub Actions**
3. Push changes to the `master` branch
4. The site will automatically build and deploy

The site will be available at: `https://ppatierno.github.io/`

## Theme

The Massively theme is located in `themes/massively/`. The theme includes:
- Responsive design
- Blog post layout
- Contact form
- Social media integration

## License

The Massively theme is licensed under the [Creative Commons Attribution 3.0 License](https://html5up.net/license).
