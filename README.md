# pallab.codes

Terminal-styled technical blog built with [Zola](https://www.getzola.org/) and the [Terminimal](https://github.com/pawroman/zola-theme-terminimal) theme.

## Local Development

```bash
# Install Zola (macOS)
brew install zola

# Serve locally with hot reload
zola serve

# Build for production
zola build
```

## Deployment

Automatically deployed to GitHub Pages via GitHub Actions on push to `main`.

## Writing Posts

Create a new `.md` file in `content/`:

```markdown
+++
title = "Your Post Title"
date = 2024-12-18
description = "Brief description for SEO"
[taxonomies]
tags = ["tag1", "tag2"]
+++

Your content here...
```

## License

Content © Pallab. Theme © [Terminimal](https://github.com/pawroman/zola-theme-terminimal).
