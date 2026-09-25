# ksau-blogs

Sauraj's engineering blog - deep dives and complete guides on Kubernetes,
containers, networking and DevOps.

Built with the [Stygian](https://github.com/ksauraj/stygian) Jekyll theme in
blog mode and published to GitHub Pages at https://blog.ksauraj.eu.org.

## Local development

Requires Ruby 3.2+ and Bundler.

```bash
bundle init
bundle add jekyll
bundle exec jekyll serve --config _config.local.yml
```

For local preview the theme is installed as a gem; on GitHub Pages it is
resolved via `remote_theme: ksauraj/stygian`. Add a `_config.local.yml` that
overrides `theme: stygian` so the build finds the path-installed gem.

## Posts

Write a new post as `_posts/YYYY-MM-DD-slug.md`:

```markdown
---
layout: blog
title: "My Post"
date: "2026-09-01"
excerpt: "One-line summary shown in the list, RSS and search."
tags: ["kubernetes", "devops"]
---

Body goes here.
```

The sidebar sorts newest first and the feed, archive and home page update
automatically. Set `feed_exclude: true` in front matter to skip a post in the
RSS feed.