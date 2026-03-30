# OTN Glider Documentation

This repository hosts documentation built with MkDocs and the Material for MkDocs theme.

Getting started

- Prerequisites: Python 3.9+ and pip
```bash
python3 -m venv .venv
source .venv/bin/activate
```
- Install dependencies:
    - pip install -r requirements.txt
- Serve locally (auto-reload on changes):
    - mkdocs serve
- Build static site to the `site/` folder:
    - mkdocs build --clean

Deploying to GitHub Pages

- Ensure the repository has GitHub Pages enabled (Settings → Pages → Build and deployment: GitHub Actions)
- Push to the default branch (e.g., `main`). The included workflow `.github/workflows/gh-pages.yml` will build and
  publish the site automatically.

Project structure

- mkdocs.yml — site configuration
- docs/ — Markdown content and assets
    - index.md — Home page
    - how/ — How-to guides
    - gdp/ — Glider Data Pipeline section
    - javascripts/mathjax.js — MathJax config for LaTeX support

Useful commands

- mkdocs new . # create a new project (already done here)
- mkdocs serve # start the live-reloading docs server
- mkdocs build # build the documentation site
- mkdocs gh-deploy # deploy to GitHub Pages (alternative to the workflow)