---
title: "Hide Fzf Files"
date: 2024-01-03T16:56:22+02:00
draft: false
---

# Hide Fzf Files
To hide files from fzf, create a `.ignore` in the root of your project and add the files you want to ignore.
For example, if you want to ignore `node_modules` and `vendor` directories, add the following to `.ignore`:
```
node_modules
vendor
```

The command to use as your FZF_DEFAULT_COMMAND so that it will respect the `.ignore` file is:
```
rg --files --no-ignore-vcs --hidden
```

This will only ignore files in the `.ignore` file. Files in the .gitignore file will still be shown.

Im using ripgrep as my default search tool. Im not sure if this works with other search tools.
