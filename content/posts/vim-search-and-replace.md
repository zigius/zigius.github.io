---
title: "Vim Search and Replace"
date: 2023-05-07T17:41:39+03:00
draft: false
---

# Search and replace text in Vim buffers

To search and replace text in Vim using regexes you can use one of the following
options: 

1. Using normal mode
  ```ruby
  :%g/Page \d\+/#\d\+ Highlight/norm! ^f#r
  ```
2. Normal search and replace
  ```ruby
  :%s/\(Page \d\+\) \(#\d\+ Highlight\)/\1\r\2/g
  ```
3. Vim very magic mode
  ```ruby
  :%s/\v(Page \d+) (#\d+ Highlight)/\1\r\2/g
  ```

I personally like the third option as it is the easiest to use. 
The second option using normal search and replace might be the most obvious, but 
adds a lot of clutter. The first option using normal mode might be the most
powerful but I feel like it will take time mastering and has a steeper learning
curve.

Example text: 
```markdown
Page 66 #1 Highlight
With a Second Brain as a shield against the media storm, we no longer have to react to each idea immediately, or risk losing it forever. 
We can set things aside and get to them later when we are calmer and more grounded
```

Will be replaced with: 
```markdown
Page 66
#1 Highlight
With a Second Brain as a shield against the media storm, we no longer have to react to each idea immediately, or risk losing it forever. 
We can set things aside and get to them later when we are calmer and more grounded
```

## Extended explanation of commands

### first command using normal mode

Let's break down this command:

* `%`: Specifies that the global command should be run on the entire buffer.
* g: Starts the global command.
* `/Page \d\+/#\d\+ Highlight/`: Specifies a regular expression pattern to search for. 
  This pattern matches the text "Page", followed by one or more digits (\d\+), 
  followed by the "#" symbol, followed by one or more digits (\d\+), followed by the text "Highlight".
* `norm! ^f#r`: Specifies a normal mode command to execute on each matching line. 
  This command moves the cursor to the beginning of the line (^), 
  searches forward for the "#" symbol (f#), and replaces it with a newline character (r).

### second command search and replace

This command uses Vim's substitute command (:s) to replace each matching pattern with two lines. Here's how it works:

* %: Specifies that the substitute command should be run on the entire buffer.
* s: Starts the substitute command.
* `/\(Page \d\+\) #\(\d\+ Highlight\)/`: Specifies a regular expression pattern to search for. 
This pattern matches the text "Page", followed by one or more digits (\d\+), followed by the "#" symbol, 
followed by one or more digits (\d\+), followed by the text "Highlight". 
The \( and \) characters are used to capture groups of text, which can be referenced in the replacement text.
* /\1\r\2/g: Specifies the replacement text. This text contains two captured 
groups (\1 and \2) separated by a newline character (\r).

After running this command, each line in the buffer that matches the specified pattern will 
be split into two lines, just like in the previous solution. The first line will contain the 
text "Page" followed by the page number, and the second line will contain the highlight number and text "Highlight".

### third command very magic mode

You can use the "very magic" mode in Vim's regular expressions to avoid the need to escape most special characters, including the parentheses. 

Here's a command that uses "very magic" mode:

```
:%s/\v(Page \d+) #(\d+ Highlight)/\1\r\2/g
```

In this command, the `\v` sequence at the beginning of the pattern tells Vim to enter "very magic" mode. In this mode, most special characters, including parentheses, don't need to be escaped. 

The regular expression pattern itself matches the same text as before: "Page ", followed by one or more digits (denoted by `\d+`), followed by a space, a "#" symbol, one or more digits, another space, and the word "Highlight". The capture groups are denoted by the parentheses, but they don't need to be escaped in "very magic" mode.

The rest of the command is the same as before. It replaces each match with the first capture group (`\1`), a newline character (`\r`), and the second capture group (`\2`).

In Markdown format, the command would look like this:

```
:%s/\v(Page \d+) #(\d+ Highlight)/\1\r\2/g
```

There are of course a lot of other ways you can modify text in vim, those are just some of the methods I wanted to cover.. 
If you can think of A better option to achieve the same result please let me know :) 

