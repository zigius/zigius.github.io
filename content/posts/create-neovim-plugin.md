---
title: "Create Neovim Plugin"
date: 2023-01-14T15:57:29+02:00
draft: false
---

> **_DISCLAMER:_**
This blog post will be updated every time i improve the workflow I use when creating plugins


I recently started working on creating some plugins in neovim. In this post I will show 
you how I created a simple plugin to open the current file in Obsidian

## getting started
First I created a github repo for my plugin. The name I gave my repo is `open-in-obsidian.nvim`
[here](https://github.com/zigius/open-in-obsidian.nvim) is the link to the repo if you
simply want to view the code.

After creating the repository I created my folder structure. 

```
lua/
  obs/
    init.lua
    utils.lua
  plugin/
    init.lua
  tests/
    test_utils.lua
```

Now that the folder structure is in place we can start working on the main file of the plugin: `lua/obs/init.lua`
```lua
local M = {}

M.open_in_obsidian = function()

  local relative_path = require"obs.utils".get_git_relative_path()
  local vault = require"obs.utils".get_git_root_folder()
  -- !open "obsidian://open?vault=notes&file=second_brain/resources/the_happy_sleeper"
  -- temp path
  -- relative_path = "second_brain/resources/the_happy_sleeper.md"

  local cmd = "!open \"obsidian://open?vault=" .. vault .. "&file=" .. relative_path .. "\""
  vim.api.nvim_command(cmd)
end

-- M.setup({ vault = "notes" })
-- M.open_in_obsidian()

return M
```

Lets go over the code.
A common practice when creating lua modules is to create a local table `M`, add the functionallity
you wish to expose as functions of the table, and return it. example:
```lua
local M = {}

M.open_in_obsidian = function()
  print("hello from obsidian plugin")
end

return M
```

I used the same practice in this plugin.

The `open_in_obsidian` method is the method doing the heavy lifting. It starts by calling two other functions, `get_git_relative_path()` and `get_git_root_folder()`, both 
of which are defined in a module called `obs.utils`, which we will go over later.
The `get_git_relative_path()` function returns the path of the current file relative to the git folder, and `get_git_root_folder()` returns the name of the root git folder.
The relative path is used as the file name we would like to open. 
The root folder is the name of the [vault](https://help.obsidian.md/Getting+started/Create+a+vault) in obsidian.

The function then constructs the [Obisidian URL](https://help.obsidian.md/Advanced+topics/Using+obsidian+URI) to open the file in obsidian, and wraps it in a vim open command.
an example of the output will look something like this:
```sh
!open "obsidian://open?vault=vault_name&file=folder1/folder2/file.md"
```
The command string is then passed to vim.api.nvim_command(cmd), which will execute the command as an Ex command in Neovim.

The plugin folder contains the code that creates the user command which we will be able to execute from neovim. From what i could gather as a best practice
to writing neovim plugins, the plugin file is where you define the user commands and keybindings of the plugin.
Here is how the code looks: 
```lua
-- Register the command in your plugin file
vim.api.nvim_create_user_command("Obisidian", function()
  require"obs".open_in_obsidian()
end, { nargs = 0 })
-- vim.api.nvim_create_user_command("SummarizeArticle", SummarizeArticle, { nargs = 1})
```

We create a user command called `Obsidian` that requires our main module and
calls the open in obsidian function. 
I also created two util functions to help me construct the url to pass to obsidian. 
```lua
M.get_git_relative_path = function()
  -- Get the full path of the current file
  local file_path = vim.api.nvim_call_function("expand", {"%:p"})

  -- Get the path of the root git repository
  local git_root_path = vim.api.nvim_call_function("system", {"git rev-parse --show-toplevel"}):gsub("\n", "")
  local relative_path = string.sub(file_path,string.len(git_root_path)+2)
  return relative_path
end

M.get_git_root_folder = function()
  -- Get the path of the root git repository
  local git_root_path = vim.api.nvim_call_function("system", {"git rev-parse --show-toplevel"}):gsub("\n", "")
  local git_root_folder = string.match(git_root_path, "([^/]+)/?$")
  return git_root_folder
end
```

The first function `get_git_relative_path` gets the relative path of
the current file in the context of its git repository. The function first gets
the full path of the current file by calling the Neovim API function
`nvim_call_function` with the argument `expand` and the parameter `%:p`, which
is a Vim special variable that expands to the current file path. Then, it gets
the path of the root git repository by calling the Neovim API function
"nvim_call_function" with the argument "system" and the parameter "git rev-parse
--show-toplevel". The returned value is processed by removing newline characters
with the gsub function. Then, it calculates the relative path by taking a
substring of the current file path starting from the length of the git root path
+2. Finally, it returns the relative path.

The second function `get_git_root_folder` uses almost the same functionallity to
get the root folder, which in Obsidian terms is the name of the vault.
You can check out the tests folder to see how I checked that the functions work
as I expected them to.

That's about it. I will continue updating this blog post and repo as I find some
more tips and tricks on how to create neovim plugins so stay tuned.
