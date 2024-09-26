(website)=
# Setting up the website

The current [SNUFA website](https://snufa.net) uses GitHub page built with a Jekyll template. This is super low maintenance. If you want a slightly nicer website, like the one this guide is on, I would now recommend using [MyST](https://mystmd.org).

## GitHub pages

We set up a GitHub organisation (in our case ``snufa``) and then create a main website repository. To do this, create a repository in that organisation called ``[orgname].github.io``. This will then create that as a website ``https://[orgname].github.io``. For more information, see [GitHub pages guide](https://pages.github.com/). You can also create a custom domain name (we use ``snufa.net``). See [here](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site) for documentation.

## Jekyll

You can see [our repository here](https://github.com/snufa/snufa.github.io).

In your website repository, go to Settings, then Pages, and set Source to "Deploy from a branch" and select your main branch. You can customise it a little by creating a file ``_config.yml`` that looks like this:

```yml
theme: jekyll-theme-cayman
title: "SNUFA"
description: "Spiking Neural networks as Universal Function Approximators"
show_downloads: false
github:
  is_project_page: false
```

Then just create an ``index.md`` file in Markdown that looks something like this:

```markdown
## SNUFA

SNUFA is blah blah blah...

### SNUFA 2024

This year's workshop is ...

### Archive

* [SNUFA 2023](/2023)
* ...
```

## MyST

This has a bit more learning to get started, but allows for a much nicer website.

Start by [installing MyST](https://mystmd.org/guide/installing).

Enable GitHub pages by going to the repo Settings, then Pages, and set Source to "GitHub Actions".

Then follow the steps to set up [MyST deploment](https://mystmd.org/guide/deployment-github-pages).

Take a look at [this repository](https://github.com/snufa/guide-to-running-snufa) to get an idea of what that looks like.