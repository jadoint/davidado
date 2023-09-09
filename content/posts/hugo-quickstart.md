+++
title = 'Hugo Quickstart'
date = 2023-09-03T10:51:19-07:00
draft = false
+++

I've been dabbling with static site generators and I've been impressed with Hugo as it's fairly straightforward. It only took a few days for me to convert the majority of my static sites to use Hugo and deploy them on Cloudflare which really showcases its ease of use. I'm just leaving my reference sheet below for when I inevitably forget how to use it again a few months from now.

## Installation

**Linux**

```
sudo apt-get install hugo
```

## Create a new project

`hugo new site my-hugo-site`

You can find a theme at [themes.gohugo.io](https://themes.gohugo.io/). I'll be using the [Anatole theme](https://themes.gohugo.io/themes/anatole/)

```
cd my-hugo-site
git init
git submodule add https://github.com/lxndrblz/anatole.git themes/anatole
git submodule update --init --recursive
```

## Configuration

The configuration file is in `hugo.toml` and sample settings are below.

```
baseURL = 'https://davidado.com/'
languageCode = 'en-us'
theme = 'anatole'
title = 'David Ado'

summarylength = 20
enableEmoji = true
enableRobotsTXT = true

# Google Analytics
googleAnalytics = "G-**********"

# Syntax highlighting
pygmentsUseClasses = true   
pygmentsCodeFences = true
pygmentsCodefencesGuessSyntax = true

[params]
title = 'David Ado'
author = 'David Ado'
profilePicture = "/images/profile.jpg"
enableEmoji = true
enableRobotsTXT = true
readMore = true
relatedPosts = true
numberOfRelatedPosts = 3
description = """
Full Stack Developer
Cloud Architect
"""

## Social links
# use 'fab' when brand icons, use 'fas' when standard solid icons.
[[params.socialIcons]]
icon = "fab fa-linkedin"
title = "Linkedin"
url = "https://www.linkedin.com/in/davidado/"

[[params.socialIcons]]
icon = "fab fa-github"
title = "GitHub"
url = "https://github.com/jadoint"

[menu]

  [[menu.main]]
  name = "Home"
  identifier = "home"
  weight = 100
  url = "/"

  [[menu.main]]
  name = "Posts"
  weight = 200
  identifier = "posts"
  url = "/posts/"

  [[menu.main]]
  name = "About"
  weight = 300
  identifier = "about"
  url = "/about/about-me"
```

---

**Troubleshooting**

Images, favicons, and fonts go into the */static* directory and is referenced in the url as if the file was found in *root*. So placing *profile.jpg* in */static* would make it accessible at *localhost:1313/profile.jpg*. You can make subdirectories in */static* such as */static/images* which would make your image accessible at *localhost:1313/images/image1.jpg*.

---

## Run server

`hugo server`

## Create a post

`hugo new posts/hello-world.md`

## Create a GitHub repository

Create a new GitHub repository at [repo.new](https://repo.new/) then run the following commands within your project directory.

```
git remote add origin https://github.com/<your-gh-username>/<repository-name>
git branch -M main
git push -u origin main
```

## Deploy to Cloudflare Pages

1. Login to your Cloudflare dashboard.
2. Go to *Workers & Pages > Overview > Create application > Pages > Connect to Git*
3. Select your repository and set your build settings.
- Production branch: `main`
- Build command: `hugo`
- Build directory: `public`

## Notable themes

Good for professional blogs: [Anatole](https://themes.gohugo.io/themes/anatole/)

Blog theme with a focus on images: [Blist](https://themes.gohugo.io/themes/blist-hugo-theme/)

Good for company sites: [Up Business Theme](https://themes.gohugo.io/themes/up-business-theme/), [Spectral](https://themes.gohugo.io/themes/spectral/), [Hugo Fresh](https://themes.gohugo.io/themes/hugo-fresh/)

Good for portfolios: [Creative portfolio](https://themes.gohugo.io/themes/hugo-creative-portfolio-theme/)

Good for galleries: [Gallery](https://themes.gohugo.io/themes/hugo-theme-gallery/), [Gallery Deluxe](https://themes.gohugo.io/themes/gallerydeluxe/)
