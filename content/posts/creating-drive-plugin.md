---
title: "Embedding an image in a markdown file using google api and luasnip"
date: 2022-12-10T02:58:16+02:00
draft: false
---

Im currently working on building my second brain. I use neovim for everything I can. 
Seems like a no brainer to also try and use neovim as my notetaking app. 
A second brain consists of notes, images, videos, and much more. 
This post will focus on my workflow for adding images to markdown files. 

## Storing images
I store my notes in a github repo, and of course also locally on my laptop and phone. 
Storing images in the same way would be a bit combersome. As an alternative i decided to store my second brain 
images in my google drive and use the same format and folder structure.

My workflow goes like this: 
1. Take a screenshot of something i want to take note of. 
2. Use macos snipping tool to save the screenshot in my drive.
3. go to google drive web client.
4. copy the link to the new image. the link is in the following format: https://drive.google.com/file/d/10GTx_K7hrtphNbJYV6bB-A9tqYbPEK3k/view?usp=share_link
5. take the id from the drive link:  `10GTx_K7hrtphNbJYV6bB-A9tqYbPEK3k`
6. create an embedded image link: 
```md
![image text](https://drive.google.com/uc?id=10GTx_K7hrtphNbJYV6bB-A9tqYbPEK3k)
```

This was ok to do at first, but it became annoying to perform so many steps everytime i want to add an image to a note

I decided to take a couple of steps to improve my workflow.

## First try at improving the worflow

I decided to create a lua snippet or a saved vim command that takes the contents in the clipboard (the last entry) and extracts the id of the entry. 
then it should paste an image tag for markdown 
(`![example image](https://drive.google.com/uc?id=1-tHUz2-5op1EBiJhZgpbnsTSpH2WjsLk)`) 
with the id of the image
This will make my flow much shorter. Steps 5-7 will be replaced with a simple snippet

I started with creating my lua function: 
```lua
local drive_image = function()
  local register = vim.fn.getreg('+')
  local index, j = string.find(register, "/d/(.-)/")
  if index == nil then return { "" } end
  local id = string.sub(register, index + 3, j - 1)
  return { "(https://drive.google.com/uc?id=" .. id .. ")" }
end
```

We define a function `drive_image` that gets the contents of the unnamed register `+`.
then we use `string.find` lua function to find the id in the link with a regex.
we trim the unnessecary slashes and the extra `/d` from the regex results and return 
the link we need to use, to embed the image in the markdown file.

Next, I needed to define my luasnip snippet:

```lua
local status_ok, ls = pcall(require, "luasnip")
if not status_ok then
  return
end

local snip = ls.snippet
local t = ls.text_node
local i = ls.insert_node
local func = ls.function_node

require("luasnip.loaders.from_vscode").lazy_load()
require("luasnip.loaders.from_snipmate").lazy_load()

ls.config.set_config {
  history = true,
  updateevents = "TextChanged,TextChangedI",
}

local drive_image = function()
  local register = vim.fn.getreg('+')
  local index, j = string.find(register, "/d/(.-)/")
  if index == nil then return { "" } end
  local id = string.sub(register, index + 3, j - 1)
  return { "(https://drive.google.com/uc?id=" .. id .. ")" }
end

ls.add_snippets(nil, {
  all = {
    snip({
      trig = "drive-image",
      namr = "drive",
      dscr = "Inserts image from drive in markdown format",
    }, {
      t("!["), i(1, "image text"), t("]"), func(drive_image, {}),
    }),
  },
})
```

I use luasnip as my snippet engine. I left only the relevant parts in the code. 
Basically we create a snippet that when invoked with `drive-image` will call my function
`drive_image` and surround the returned result with: `![image text](<RETURNED RESULT>)`, placing the 
cursor on the `i` of `image text` so we can replace the image text with a text of our liking.

This improves the workflow. the steps needed are reduced: 
1. Take a screenshot of something i want to take note of. 
2. Use macos snipping tool to save the screenshot in my drive.
3. go to google drive web client.
4. copy the link to the new image. the link is in the following format: `https://drive.google.com/file/d/10GTx_K7hrtphNbJYV6bB-A9tqYbPEK3k/view?usp=share_link`
5. use luasnippet with name `drive-image`

Instead of building the image markdown manually, which was a hassle, I can now use my snippet. 
The major bottleneck has been removed but i wanted to take it further

## Second try at improving the worflow
Leaving my editor to copy the link to the image can drastically increase the time it takes me to create a note with an image.
Notes should be created with haste to so as they will not interfere with my current task.
I decided to use google drive API to retrieve the id of the last image uploaded to my drive. 
I wanted to create a lua command or snippet that will use the API to get the id, instead of having to manually copying the link.
I tried finding an existing google drive cli tool, but all were unmaintained and broken, so I created my own. 

### Create a google developer application.
First, I needed to create a google cloud application.
I visited the [google developer console](https://console.cloud.google.com/) and created my app.
![Creating a developer app](https://drive.google.com/uc?id=10PcFGZwO6t76njYrraIN_IlOavJ1DJGw)

![Developer console homepage](https://drive.google.com/uc?id=10QSiFHL3WBKDmRxL_Vx5isHaTNmKNa5m)

Next, we need to enable apis and services: 
![image text](https://drive.google.com/uc?id=10XQeoZ9Zo79-1dou1Jwk3LVq-k4PJIaU)

After creating the app go to your [apis and services dashboard](https://console.cloud.google.com/apis/dashboard?project=hugo-drive-to-markdown) page and
click on `ENABLE APIS AND SERVICES` button, located at [this link](https://console.cloud.google.com/apis/library/browse?project=hugo-drive-to-markdown&q=drive).
Search for google drive api: 
![image text](https://drive.google.com/uc?id=10Xe5r9KPe4xb1We0Qb9oGbT-GpOrchqR)
and enable it:
![Enable](https://drive.google.com/uc?id=10Y_pX1Nh2VFWrjTWrC0Bhk7jUSCjVlFZ)
Now you need to create credentials for the api. Click on `CREATE CREDENTIALS` and select the drive api. We will be accessing user data so select that option when 
prompted to enter what data you will be accessing.

Click next
now enter your app name and app details along with your email address.
![image text](https://drive.google.com/uc?id=10Z42ENNWjaxa31A2d1rHX6idYO0MTLKQ)
![image text](https://drive.google.com/uc?id=10k_tL74b-ub3uX_R-OYRZFALTFN0q2se)
now we add scopes. The only scope the app will need is the following:
https://www.googleapis.com/auth/drive.metadata.readonly

select it and click the `update` button
![image text](https://drive.google.com/uc?id=10mp6RRT54CXkWhW4kD6WrVH8Ehotkfkh)
![image text](https://drive.google.com/uc?id=10nj6OygRINjBmZroAUKnLUzy9h0TmpeD)
![image text](https://drive.google.com/uc?id=10tZH2T4fgnO4JCQFKRu5rXSonhszfhRj)

Now we need to create the OAuth client:
Select the application type to be a Web application.
Select a name for your app, and add an authorized javascript origin and redirect URI. For development
purposes we will use localhost.
![image text](https://drive.google.com/uc?id=113zWT0D_E_gDfcIbF1lZ4OSBNtdPppJ9)

click on done, and copy the client-id and download the client id and json file. We will need it soon.
Here is an example of what that file is supposed to look like:
```json
{
  "web": {
    "client_id": "<id>.apps.googleusercontent.com",
    "project_id": "<project-id>",
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "token_uri": "https://oauth2.googleapis.com/token",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
    "client_secret": "<client-secret>",
    "redirect_uris": ["http://localhost:8080/oauth2callback"],
    "javascript_origins": ["http://localhost:8080"]
  }
}
```
We are done with configuring our credentials:
![image text](https://drive.google.com/uc?id=114gvBgOC4Haxh0VIwaG1X7ZsGOvLdeTw)

> Make sure the client secret exists in the credentials file you downloaded. If you dont see the client_secret property you might need to 
reset or create it. you can reset it by going to the web credentials page we just created and clicking on create secret or alternatively, change secret:
![image text](https://drive.google.com/uc?id=11El5GJneLNT-bw-HT7HE50eOB1ovuYiC)

Another step you need to make is to approve your personal email as a developer: 
Go to the [OAuth consent screen](https://console.cloud.google.com/apis/credentials/consent?orgonly=true&project=hugo-drive-to-markdown&supportedpurview=organizationId) and add a test user:
![image text](https://drive.google.com/uc?id=11GCdF-4RDn-WInhyz_ozxxZfc9H0E9fp)
set your email as the test user email.
![image text](https://drive.google.com/uc?id=11Gti6cUobgoITWQIEAbTDS36LG5oVOK_)

Thats it! now we can create our cli app to use the credentials we just set up. 

### Create a nodejs cli app
We will be writing the application as a nodejs app.
You can find the source code for the app [here](https://github.com/zigius/drive-last-image-to-md)
I pretty much combined two tutorials when creating the app. 
The first tutorial I used was the [nodejs quickstart] tutorial.
It illustrates how you can set up and run an app that calls the google drive api. 
The second tutorial I used was [OAuth in Node.js CLI Apps](https://thecodebarbarian.com/oauth-in-nodejs-cli-apps.html) on 
how to create a command line app that uses oauth to log in to slack. I combined the two tutorials to create my own cli app.
here is the main cli_app.js code snippet: 
```js
#!/usr/bin/env node

"use strict";

import fs from "fs";
import Configstore from "configstore";
import yargs from "yargs";
import path, { dirname } from 'path';
import { fileURLToPath } from 'url';
const __dirname = dirname(fileURLToPath(import.meta.url));

import { google } from "googleapis";
import { DriveClient } from "./services/driveClient.js";
import { AuthenticationClient } from "./services/authenticationClient.js";

const config = new Configstore("drive-cli");

/**
 * To use OAuth2 authentication, we need access to a CLIENT_ID, CLIENT_SECRET, AND REDIRECT_URI.  To get these credentials for your application, visit https://console.cloud.google.com/apis/credentials.
 */

const keysFile = fs.readFileSync(path.resolve(__dirname, './oauth2.keys.json'));
let keys = JSON.parse(keysFile).web;

/**
 * Create a new OAuth2 client with the configured keys.
 */
const oauth2Client = new google.auth.OAuth2(
  keys.client_id,
  keys.client_secret,
  keys.redirect_uris[0]
);

/**
 * This is one of the many ways you can configure googleapis to use authentication credentials.  In this method, we're setting a global reference for all APIs.  Any other API you use here, like google.drive('v3'), will now use this auth client. You can also override the auth client at the service and method call levels.
 */
google.options({ auth: oauth2Client });

const argv = yargs(process.argv.slice(2))
  .command(
    "login",
    "the login command",
    () => {},
    async (argv) => {
      let authenticationClient = new AuthenticationClient(oauth2Client);
      const credentials = await authenticationClient.authenticate();
      oauth2Client.credentials = credentials; // eslint-disable-line require-atomic-updates
      config.set({ credentials: oauth2Client.credentials });
      process.exit(0);
    }
  )
  .command(
    "lastFile",
    "get last file",
    () => {},
    async (argv) => {
      let credentials = config.get("credentials");
      oauth2Client.credentials = credentials;
      let driveClient = new DriveClient(oauth2Client, google);
      let authenticationClient = new AuthenticationClient(oauth2Client);
      try {
        let id = await driveClient.getLastFileId();
        console.log(id);
        return id;
      } catch (e) {
        credentials = await authenticationClient.authenticate();
        oauth2Client.credentials = credentials; // eslint-disable-line require-atomic-updates
        config.set({ credentials: oauth2Client.credentials });

        let id = await driveClient.getLastFileId(oauth2Client);
        console.log(id);
      }
    }
  ).argv;
```

It uses yargs to create the cli app. 
All you need to do to use the app from the command line is clone the repository, copy the credentials 
json file we downloaded earlier (with the name `oauth2.keys.json`) to the root of the repository, and run the following commands:
```sh
chmod +x cli_app.js
./cli_app.js lastFile
```

It will redirect you to a login screen and once you approve the OAuth authorization screen, the file id will be printed to the screen.

### Create a luasnip that uses my cli app

To use the command line app I created from within neovim I had to create a new lua function and snippet.
Here is a snippet with the function code:
```lua
local drive_image_last = function()
  local handle = io.popen("cli_app.js lastFile")
  local result = handle:read("*a")
  handle:close()
  id = string.gsub(result, "\n", "")
  return { "(https://drive.google.com/uc?id=" .. id .. ")" }
end
```

The function calls the api, and returns the link required to embed the file in the markdown, same format as we previously saw. 
To use the API from within neovim we also need to add our app to the `PATH` environment variable:
```sh
export PATH="/Users/me/workspace/public-repos/drive-last-image-to-md:$PATH"
```

And here is the updated luasnip snippets file:
```lua
local status_ok, ls = pcall(require, "luasnip")
if not status_ok then
  return
end

local snip = ls.snippet
local t = ls.text_node
local i = ls.insert_node
local func = ls.function_node

require("luasnip.loaders.from_vscode").lazy_load()
require("luasnip.loaders.from_snipmate").lazy_load()

ls.config.set_config {
  history = true,
  updateevents = "TextChanged,TextChangedI",
}

local drive_image = function()
  local register = vim.fn.getreg('+')
  local index, j = string.find(register, "/d/(.-)/")
  if index == nil then return { "" } end
  local id = string.sub(register, index + 3, j - 1)
  return { "(https://drive.google.com/uc?id=" .. id .. ")" }
end

ls.add_snippets(nil, {
  all = {
    snip({
      trig = "drive-image-last",
      namr = "drive",
      dscr = "Inserts last image from drive in markdown format",
    }, {
      t("!["), i(1, "image text"), t("]"), func(drive_image_last, {}),
    }),
    snip({
      trig = "drive-image",
      namr = "drive",
      dscr = "Inserts image from drive in markdown format",
    }, {
      t("!["), i(1, "image text"), t("]"), func(drive_image, {}),
    }),
  },
})
```

Now that we are done the workflow is so much easier! only thing we need to do to add the image to our note is this:
My workflow goes like this: 
1. Take a screenshot of something i want to take note of and use macos snipping tool to save the screenshot in my drive.
3. Use `drive-image-last` snippet to embed the last drive image to the markdown file

Hope you enjoyed 😎
