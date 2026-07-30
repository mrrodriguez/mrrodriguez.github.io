# metasimple.org Blog Repository

This repository contains the source code for the GitHub Pages blog hosted at [metasimple.org](https://metasimple.org) (mrrodriguez.github.io).

## Environment Expectations

This project explicitly requires Ruby to run the site locally.
- **Ruby Version:** The expected version is defined in the `.ruby-version` file (currently `3.3.6`). Environment managers like `mise`, `rbenv`, or `rvm` will automatically detect this and switch to the correct version.
- **Dependency Management:** Gems are managed via Bundler. The `Gemfile` uses `ruby RUBY_VERSION`, which tells Bundler to expect the Ruby version that is currently active (set by `.ruby-version`). 
- **GitHub Pages Sync:** We use the `github-pages` gem to mirror the environment used by GitHub Pages' automated build system. This ensures that the local build behaves exactly like the live site.

## Local Development

The site is built with Jekyll. To preview changes locally with live reloading, follow these steps:

### Prerequisites
Make sure you have Ruby and Bundler installed. If using `mise`, ensure you have installed the Ruby version specified in `.ruby-version` (`mise install`).

1. **Install dependencies:**
   ```bash
   bundle install
   ```

2. **Run the local development server (with live reloading):**
   ```bash
   bundle exec jekyll serve --livereload
   ```

3. **View the site:**
   Open your browser and navigate to `http://127.0.0.1:4000/`. When you save changes to posts or layouts, the browser will automatically refresh.

## Dependency Upgrades

To keep the local environment up-to-date with the latest GitHub Pages versions and security patches, you should periodically update the gems.

1. **Update the GitHub Pages gem (and other dependencies):**
   ```bash
   bundle update
   ```
   *To only update the GitHub Pages gem, you can run `bundle update github-pages`.*

2. **Commit the new lockfile:**
   After upgrading, `Gemfile.lock` will have changes. Commit these changes to ensure the locked versions are tracked in Git.

*Note: This `README.md` file is explicitly excluded in `_config.yml` so it will not be generated as a page on the live site.*
