# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal technical blog built with Jekyll 4.4 and the Minima theme, hosted via GitHub Pages.

## Development Commands

```bash
# Install dependencies
bundle install

# Serve locally with live reload (available at http://localhost:4000)
bundle exec jekyll serve

# Build the site (output to _site/)
bundle exec jekyll build
```

Note: changes to `_config.yml` require restarting the server.

## Architecture

- **Theme**: Minima ~2.5, with a wider content wrapper override in `assets/css/style.scss`
- **Custom post layout**: `_layouts/post.html` extends the default layout and injects Mermaid.js (loaded from CDN) to render `mermaid` fenced code blocks as diagrams
- **Category archives**: Enabled via `jekyll-archives` plugin — posts use `categories` in front matter (e.g., `categories: [java, macOS]`) and archives are generated at `/categories/:name/`
- **Plugins**: `jekyll-feed` (RSS), `jekyll-archives` (category pages)

## Writing Posts

Posts live in `_posts/` with the naming convention `YYYY-MM-DD-slug.md`. Front matter requires:

```yaml
---
layout: post
title: "Post Title"
date: YYYY-MM-DD HH:MM:SS +0530
categories: [category1, category2]
---
```

Mermaid diagrams work in posts by using a `mermaid` fenced code block — the custom post layout handles rendering.
