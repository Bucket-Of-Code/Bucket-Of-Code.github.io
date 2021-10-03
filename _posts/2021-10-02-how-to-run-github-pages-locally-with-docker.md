---
layout: post
permalink: /how-to-run-github-pages-locally-with-docker
---

Here is a quick and easy way to test a [Github Pages](https://pages.github.com/) site locally using the [Jekyll Docker image for Github Pages](https://hub.docker.com/r/jekyll/jekyll/tags?name=pages&page=1&ordering=last_updated). This allows you to see and test your changes to your Github Pages site before pushing them to Github. This solution only requires Docker to be installed and does not require you to install Ruby to run Jekyll. Docker is a pretty common technology used these days to be able to run applications locally. And as a developer, you are more likely to have Docker than Ruby installed on your machine...unless, of course, you are a Ruby developer.

## Prequisites
- [Docker](https://www.docker.com/products/personal)

That's it!

This approach doesn't require Ruby or [Jekyll](https://jekyllrb.com/). It doesn't even require a GitHub repository for your site. However, it would be nice to have a known, working site on Github Pages, so that you reduce the amount of troubleshooting you may need to do to get this solution working.

## A Basic `docker-compose` File
In its simplest form, here is a `docker-compose.yml` file to get you started.

```yaml
# docker-compose.yml
version: "3.9"
services:
  jekyll:
    image: jekyll/jekyll:pages
    command: jekyll serve
    ports:
      - 4000
    volumes:
      - .:/srv/jekyll
```

Using this `docker-compose.yml` file will:
- use the official `jekyll/jekyll:pages` Docker image
- render only pages that have been published and not in the future
- render the site into a `_site` directory in your project
- make your site available in a browser at `localhost:4000`

This file should be enough to get you started. Now you just have to run `docker-compose up` and visit your site in a browser at `http://localhost:4000`.

## A More Advanced `docker-compose` File
Whenever I start a new Github Pages site, I always start with the `docker-compose.yml` file below. It's based off of the file above, but it adds a few more features that I was looking for while writing content.

```yaml
version: "3.9"
services:
  jekyll:
    image: jekyll/jekyll:pages
    command: >
      jekyll serve \
      --watch \  # Enable auto-regeneration of the site when files are modified.
      --force_polling \  # Force watch to use polling
      --drafts \  # Process and render draft posts
      --unpublished \  # Render posts that were marked as unpublished
      --future \  # Publish posts or collection documents with a future date
      --verbose \ # Print verbose output
      --trace \  # Show the full backtrace when an error occurs
      --strict_front_matter \  # Cause a build to fail if there is a YAML syntax error in a page's front matter
      -d /tmp/_site  # Change the directory where Jekyll will write files
    ports:
      - 4001:4000  # Serves the site on port 4001. Change this if you have other apps running on that port
    # A simple healthcheck to tell Docker that the site is up and running
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000"]
      interval: 1m30s
      timeout: 10s
      retries: 3
    volumes:
      - .:/srv/jekyll  # Mount the project into the pwd of the Docker container
```

I've commented on each line of the file to show what each does. In a nutshell, this more advanced docker-compose file will:
- use the official `jekyll/jekyll:pages` Docker image (link)
- render all pages, regardless of publishing status
- only render the site in the Docker container and not in your project
- make your site available in a browser at `localhost:4001`
- watch for any changes to your files and re-render the site
- provide more information when rendering your site and when errors occur

## Troubleshooting
This method isn't 100% foolproof and does have some issues. Here are some FAQs that I think might be useful.

### Why are my changes not reflected on my site, when I add a new page or make changes to my content?
The `--watch` option does its best to determine when relevant files have changed. However, Jekyll's internal cache sometimes gets in the way. The cleanest way to remove the cache is to recreate the Docker container.

#### Recreate the Docker container
1. `ctrl-c` (on Windows/Linux) to shut down the container when it is running. If it hangs, try `ctrl-c` again to force it to shut down. If this still doesn't work, try restarting the Docker service.
2. Run `docker-compose down` this will stop the container if it's still running and delete the container completely
3. Run `docker-compose up` to recreate the contatiner.

### Why are my changes not reflected on my site, when I make changes to my `_config.yml` file?
Unfortunately, the `_config.yml` is excluded from the `--watch` flag. The only way to re-read the config file is to stop and restart the container. If this still doesn't work, try recreating the container with the instructions above.

### When I visit the site in my browser, the site cannot be reached.
The default port for Jekyll is port 4000 and the output in the console shows `http://0.0.0.0:4000` for the URL. However, that port is is exposed only inside the container. The basic docker-compose file above uses port 4000, but the advanced file exposes the site on port 4001.

Here are a few things to check:
- Make sure your Docker container is actually running and that the console shows `Server address: `
- Use `localhost` instead of `0.0.0.0` in your browser
- Make sure your `docker-compose.yml` file exposes the port
- Make sure the port you use in the browser matches the port you have exposed in your `docker-compose.yml` file (port `4001` in the case of the advanced file above).