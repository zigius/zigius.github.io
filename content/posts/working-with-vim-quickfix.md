---
title: "Search and replace in vim"
date: 2023-02-18T15:35:17+02:00
draft: false
---

## Why use the quickfix list?
Recently I started using the quickfix list in vim, a hidden gem in the vim
ecosystem. The main reason I started working with the qf list is I wanted a way
to search and replace text in multiple files in my project. 

## getting started - search
I use telescope and FZF as my search plugins, but they do not provide 
replace functionality out of the box. They do however, support sending all or a
selected search results to the quickfix list. To send all results of `telescope grep_string` 
picker to quickfix, you can use the C-Q keyboard shortcut. 
To send selected results to quickfix list, select the items you want using TAB, and press ALT-Q to send the selection to the quickfix.
In FZF, you can select entries in the same way, using the TAB key, and send them
to the quickfix list with CTRL+T.

## getting started - replace
Now that we have the results populated in the qf list, we need a way to replace
the text. We will use the cdo command. 
To replace text in items from quickfix, use the following command:

```
:cdo s/foo/bar
```
`cdo` runs the command on all items in the quickfix list. For a more in depth
guide you can view the following [blog post](https://chrisarcand.com/vims-new-cdo-command/)
 
## search and replace - advanced configurations.
I wanted a way to remove items from the qf list. I found a [plugin](https://github.com/romainl/vim-qf) 
that does exactly what I need. The `vim-qf` plugin enhances the qf list with
additional features and mappings. One of the cool features of the plugin 
is the `Reject` command. You can remove the current result from the qf list 
by running the following command: 
`:.Reject`

Using the Reject command, I can optimize the results of the search query 
without having to search using a fuzzy finder again.

I created a ftplugin for the quickfix list, and mapped `dd` to reject files
from the list.
filename: `/Users/myusername/.vim/after/ftplugin/qf.lua`
```lua
vim.keymap.set('n', 'dd', ':.Reject<CR>', {buffer = true})
```

Now, when in the qf list, I can press dd and it will remove the file from the
list.

## Advanced configuration 2 - search in qf files.
Another great usage that I found for the quickfix list - searching for multiple 
text strings in files. Lets say that I am searching for a file that contains the
words `foo` and `bar`, but the words are not in the same line. Telescope and fzf
will not be able to provide me with files containing both of the strings.
Lets view an example text:
```txt
foo


bar


foobar
```
And lets see the results given to us by searching for foo and bar using fzf:
![Search foo bar](https://drive.google.com/uc?id=13905HLn-_hW47cyyHYt1Xc5Ce8LkWibF)

So how can we search for files containing foo and bar?
Use the qf list 😎
First, we will populate the qf list with the results of searching for foo as
described above. 
Now that we have all the files containing the string foo in the qf list, how can 
we search those files for the string bar? 

We can use a custom telescope function I created to search the files in the qf list: 
```lua
require('telescope').setup {}

-- search in quickfix list items
vim.keymap.set('n', '<leader>mq', function()
  local set = {}
  local items = {}
  for _, item in ipairs(vim.fn.getqflist()) do
    if not set[item.bufnr] then
      set[item.bufnr] = true
      table.insert(items, { file_name = vim.api.nvim_buf_get_name(item.bufnr) })
    end
  end
  local list = {}
  for i, item in ipairs(items) do
    list[i] = item.file_name
  end
  -- vim.pretty_print(list)

  builtin.grep_string({
    -- cwd = '~',
    search = "",
    search_dirs = list,
  })
end, opts)
```
This Lua function sets a key mapping for the normal mode in Vim. The key mapping is <leader>mq, 
which means the leader key (usually the backslash \ key) followed by the letter "m" and then the letter "q".

When this key mapping is triggered, it executes an anonymous function that does the following:

Initializes two empty tables, set and items.
Uses the built-in getqflist function in Vim to get a list of quickfix items. Each quickfix 
item represents a location in a file where the string foo was found.
Iterates over the list of quickfix items and creates a new table `items` that contains only the file names of the items, without duplicates.
Creates a new table list that contains only the file names of the items, in the order they were encountered in items.
Calls the telescope `grep_string` function with a table that contains the following arguments:
search: an empty string, indicating that we want to search for nothing.
search_dirs: a list of file names that we want to search in. This list is the list table we just created.
By passing it a list of qf file names, we are telling it to search for the empty string in all of those files.

Place the function in your telescope.nvim configuration file, or anywhere else
you would like. To read more about lua in neovim, check out my previous blog
post on [creating a neovim plugin](https://zigius.github.io/posts/create-neovim-plugin/)

Now we can search those files for the string `bar`

## Advanced configuration 2 - more plugins and tricks.
1. To open the qf list you need to use this command: `:copen`.
2. I also use [this plugin](https://github.com/kevinhwang91/nvim-bqf) to have a preview of the results in the qf list.

Thats it for today 😊
