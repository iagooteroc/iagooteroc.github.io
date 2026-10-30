
## Overview

These are the steps I followed to deploy this website to GitHub Pages. I used **Astro** as the web framework, which provides [official instructions](https://docs.astro.build/en/guides/deploy/github/) on how to do this.

---
## **Step 1** -- GitHub Action

Create a new file in your GitHub repo named `.github/workflows/deploy.yml` and paste this:
```yml
name: Deploy to GitHub Pages

on:
  # Trigger the workflow every time you push to the `main` branch
  # Using a different branch name? Replace `main` with your branch’s name
  push:
    branches: [ main ]
  # Allows you to run this workflow manually from the Actions tab on GitHub.
  workflow_dispatch:

# Allow this job to clone the repo and create a page deployment
permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout your repository using git
        uses: actions/checkout@v7
      - name: Install, build, and upload your site
        uses: withastro/action@v6
        # with:
          # path: . # The root location of your Astro project inside the repository. (optional)
          # node-version: 24 # The specific version of Node that should be used to build your site. Defaults to 24. (optional)
          # package-manager: pnpm@latest # The Node package manager that should be used to install dependencies and build your site. Automatically detected based on your lockfile. (optional)
          # build-cmd: pnpm run build # The command to run to build your site. Runs the package build script/task by default. (optional)
        # env:
          # PUBLIC_POKEAPI: 'https://pokeapi.co/api/v2' # Use single quotation marks for the variable value. (optional)

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```
---
## **Step 2** -- Set site URL

In your Astro config file (`astro.config.mjs`), set `site` to the GitHub URL of your deployed site.

```mjs
import { defineConfig } from 'astro/config'

export default defineConfig({
  site: 'https://<username>.github.io',
})
```
---
## **Step 3** -- Configure *base*

In your Astro config file (`astro.config.mjs`), set `base` to your repository name.

```mjs
import { defineConfig } from 'astro/config'

export default defineConfig({
  site: 'https://<username>.github.io',
  base: '/<reponame>',
})
```

However, you can skip this is your repository name matches the special `<username>.github.io` pattern (e.g. `https://github.com/<username>/<username>.github.io/`), which is what I did.That way I don't have to prefix all my internal page links with the `base` (e.g. `<a href="/<reponame>/about">About</a>`)

---
## **Step 4** -- GitHub Pages

On your repository's **Settings** tab, go to the **Pages** section and choose **GitHub Actions** as the **Source** of your site.

---
## Lessons learned

On new pushes to your repository, the configured GitHub Action will now deploy the new version to your GitHub Page.

#### Sources

- https://docs.astro.build/en/guides/deploy/github/