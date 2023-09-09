+++
title = 'Git Reference'
date = 2023-08-31T09:33:38-07:00
draft = false
+++

I've been using `git` for years now but there are commands I use every day while others, not so much.

## New Repository

Create a new repository on GitHub at [https://repo.new/](https://repo.new/). Then locally, run:

```
echo "# anything" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/<your-gh-username>/<repository-name>
git push -u origin main
```

## Removing something from git

Because I forgot my `.gitignore` and the node_modules directory got committed again. Create the `.gitignore` file this time and run:

```
git rm -r --cached node_modules
git commit -am "Removed node_modules directory"
```

## Committing a modified file in a submodule

`cd` inside the submodule directory then add and commit to git from there. You can then go to your project directory and add and commit to git as normal.