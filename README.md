# mohinee.github.io

Source for my personal blog at [mohinee.github.io](https://mohinee.github.io).

Built with [Jekyll](https://jekyllrb.com/) and hosted on GitHub Pages.

---

## Deployment — first time setup

These steps need to happen on your machine and your GitHub account. The whole
thing is about 10 minutes.

### 1. Create the GitHub repository

GitHub Pages has a special convention: a repo named `<your-username>.github.io`
automatically becomes a website at `https://<your-username>.github.io`.

1. Sign in to [GitHub](https://github.com).
2. Click the **+** icon → **New repository**.
3. Repository name: **`mohinee.github.io`** (or whatever your GitHub username is —
   this is exact and case-sensitive on the URL).
4. Set it to **Public** (required for free GitHub Pages).
5. Don't initialize with anything — leave README, .gitignore, and license unchecked.
6. Click **Create repository**.

GitHub will show you a page with setup commands. Keep it open.

### 2. Push this code to the repo

From the directory containing this README:

```bash
git init
git add .
git commit -m "Initial site"
git branch -M main
git remote add origin https://github.com/<your-username>/<your-username>.github.io.git
git push -u origin main
```

Replace `<your-username>` with your actual GitHub username (twice — it appears
in the URL because of the naming convention).

### 3. Enable GitHub Pages

1. Go to your new repository on GitHub.
2. Click **Settings** → **Pages** (in the left sidebar).
3. Under **Source**, select **Deploy from a branch**.
4. Branch: **`main`**, folder: **`/ (root)`**. Click **Save**.
5. Wait 1–2 minutes for the first build.

Your site will be live at `https://<your-username>.github.io`.

### 4. Update the config

Open `_config.yml` and update these fields with your actual details if any
differ from the defaults:

```yaml
url: "https://<your-username>.github.io"
github_username: <your-username>
linkedin_username: <your-linkedin-slug>
email: <your-email>
```

Commit and push the change. The site rebuilds automatically.

---

## Writing posts

Posts live in `_posts/` and follow the naming convention `YYYY-MM-DD-title.md`.

Front matter looks like:

```yaml
---
title: "Your post title"
date: 2026-05-02
excerpt: "A 1–2 sentence summary that shows on the home page."
reading_time: 7
---
```

After the front matter, write Markdown. Code blocks use triple backticks with
language hints (`` ```jsx ``, `` ```ts ``, etc.). The site uses Rouge for
syntax highlighting, which supports most languages out of the box.

To publish: commit, push to `main`, wait for the rebuild.

---

## Local preview (optional)

If you want to preview changes locally before pushing:

```bash
# One-time setup (requires Ruby)
gem install bundler
bundle install

# Run the dev server
bundle exec jekyll serve
```

Open `http://localhost:4000`. Changes to most files reload automatically;
changes to `_config.yml` require restarting the server.

---

## Structure

```
.
├── _config.yml          # Site config
├── _layouts/            # HTML templates
│   ├── default.html
│   └── post.html
├── _posts/              # Blog posts (Markdown)
├── assets/css/style.css # Styles
├── about.md             # About page
├── index.html           # Home page
└── Gemfile              # Ruby dependencies (for local preview)
```
