---
title: "How i created my blog in 30 minutes"
date: 2022-11-20T09:03:20-08:00
draft: false
---

Seems like everyone today has a blog, a vlog and a podcast.

This post will give you a hands on explanation on how to create your blog in 5 minutes, and deploy it on Github pages.

## Steps

### Create the repo

To keep things simple, we will create a github pages repository. 
For a more detailed instruction visit [this link](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site)

First, go to github and create a new repository with your name. example: `zigius/zigius.github.io`

After you create the repo you can clone it

### Create gh-pages branch
Hugo deploys its static website files to a branch called gh-pages. You need to create a new branch with that name

## Set up hugo as blog engine

1. cd to the directory of your github pages
2. install hugo. in mac: `brew install hugo`
4. run the following command
```sh
hugo new site . --force
```

This will generate a skeleton for your site
Now continue with the quickstart tutorial as explained [here](https://gohugo.io/getting-started/quick-start/)

When required to change the url in the `config.toml` file, change it to the base url of your github pages:
```toml
baseURL = 'https://zigius.github.io/'
```

### set up github pages as your hosting solution
for in depth explanation visit [this link](https://gohugo.io/hosting-and-deployment/hosting-on-github/)

All i had to do from the above mentioned tutorial was add the following file to github:
`.github/workflows/gh-pages.yml`

Next, I had to go to Repo settings -> Pages and change the branch to `gh-pages`

Thats basically it :)
