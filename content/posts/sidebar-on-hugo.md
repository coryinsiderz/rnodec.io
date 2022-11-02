---
title: "Sidebar on Hugo"
date: 2022-10-21
draft: false
---

# Our Website

Is created with hugo, leveraging a hugo theme called [ficurinia](https://gitlab.com/gabmus/hugo-ficurinia).  Tip the man.  This is what I know after a lowest-touch-possible approach of learning how to use hugo / how hugo works.  

## What is hugo

hugo is a cli tool for generating "static websites."  Meaning, there is a standard defined structure that a website directory can take, which will, in a single command, convert it from a nicely organized (relatively) structure to the full on complex web of ugly `.html` and `.xml` and `.css` etc files that comprise what the web server actually serves.   

## Start Line

Once you initialize a new website (`hugo new site rnodec.io`) in a new directory (`cd rnodec.io; git init`), hugo **themes** provide a beatiful, functional, and easily customizable website ootb by just connecting your [desired theme](https://themes.gohugo.io) as a git submodule in your `./themes` directory. Make no mistake though - the distance between starting point of a barebones hugo site and a full fledged theme is a marathon.  One does not have to know how to build a theme in order to use a theme.  

Initialize the themes' repo into your repo as a git submodule:
```
git submodule add https://gitlab.com/gabmus/hugo-ficurinia.git themes/hugo-ficurinia
```
Copy up the example `config.toml` that they gave you:
```bash
cp themes/hugo-ficurinia/exampleSite/config.toml .
```

**Important Note - these themes to choose from all will vary widely in how well they work, their intent and functionality, their bells and whistles, how they are configured and customized, how well they are doc-ed, and how supported they still are.  Choose wisely**

Once you are there, assuming you have a suitable local host available, is run:
```bash
hugo server -D
```
...and you can access your website at localhost:1313.  (yea, don't forget you need to install hugo)

This is the plain default starting point for the theme.  Now, you really dig into that themes' doc in order to start customizing the site to fit your needs.  

## Understanding the mess

Yeah the directories and files you see here are still messy and confusing to a non web developer like me.  But here's my mental model:

* First of all remember, the whole idea here is to build a static site from go templates
* What you see on the browser at any given time is an html file
* How that particular html file got created always starts in `themes/*your-theme*/layouts/_defaults/baseof.html`
* It has access to some context provided to it at *hugo build time*.  Much of this context comes from `config.toml`.  hugo reads this file in as a data structure and uses it everywhere.  
* `config.toml` is where you tell hugo which theme to use, along with provide all inputs to the theme itself.  
* hugo also grabs some context from the "frontof matter* in your content files.  This is like hugo-specific (theme defined) metadata that you prepend all your content files with.  
* Content files, in the case of a blog site like this (this is not a blog site; just a site with some blogs) can just be standard markdown documents.  hugo handles making sure this renders the markdown in the browser in html format (preserving site headers, footers, style, etc).  (note that all you had to do here is create the content)
* Notice that you have in your top level website dir a completely empty replica of the directory structure in the theme itself (`archetypes`, `content`, `layouts`, etc... these are the standard directories hugo expects to find).  You can override any file from the theme by just creating your own in it's place.  (or copying up the themes' version and making edits)


## How to add a section to your webpage (a useful learning exercise)

Observe that each of the directories that you define in your `./content` dir corresponds to a "type" in the world of hugo layouts residing throughout your `./themes/*your-theme*/layouts` dir. 

So if you wanted to create a new section of your website `rnodec.io/finishline`, you would start by creating the content:
```bash
hugo new finishline.md
cat >> ./content/finishline.md << EOF
No such thing
EOF
```

Now (after your website re-renders automatically (if you are running hugo server with `--disableFastRender`, hugo will completely rebuild the entire site for you on every change)), you go to `localhost:1313/finishline` in your browser and voila, you have a new section.  But how is that getting rendered exactly?  

Time to start with `baseof`.  Look at your themes' `/layouts/_defaults/baseof.html` file.  Notice all the `{{` and `}}` characters.  These denote go templates.  Inside these braces are functions and scripts that hugo is processing to replace those `{{ }}` placeholders with actual processed html code (hugo is written in go). The html that you see when you "inspect page source."  This `baseof.html` file reads in other similar go-templatted `.html` files, executes all aforementioned go scripts, and ultimately generate based on the input.  This chain can be complex, but you should be able to follow it if necessary.  

At the heart of that `baseof.html` file is a block of code that looks like this:
```go
    {{- block "main" . }}{{- end }}
```
Somehow, this block gets connected to another `.html` file.  There are a bunch of `*.html` files in the `layouts/` dir of the theme.  Which one hugo chooses to insert in that placeholder block depends on the request, and hugo is very "opinionated" about this.  See here for the lookup order that hugo follows:  https://gohugo.io/templates/lookup-order/ 

In the case of this new "/finishline" section, the order would be:  

`layouts/_default/baseof.html` -> `layouts/_default/single.html` -> `layouts/partials/single-post.html` ... you would look into `single-post.html` if you wanted to see how the `content/finishline.md` file got translated into the html version that the browser ultimately sees.  This is the flow that any page on the site will most likely take.  These are very default/magic files.  The `single.html` file is what hugo will fall back to if it can't find anything more suitable/explicit.  

> *remember this is all specific to this theme that I am using, as it's my only experience with hugo, but this should be a pretty standard pattern for hugo in general*

If I wanted to override how any of these pages render, I could make copies of those files and place them in my own `layouts/` dir.  No thanks for now though... 

If I wanted a new menu icon to show up linking to this page, I could achieve that in `config.toml` (just the doc of your theme...).  A good theme like this one should be pretty painless to work with.  

## Custom Home

I do, however, want a custom home page.  To do this, I simply copied up `layouts/index.html` from the theme, and made it my own.  

## Adding Discord Icon

The idea with this and the previous exercise is to use a task/goal to help us understand a great deal about the mystery of hugo.  In this exercise, I learned that there was no ootb support for a discord icon / link to server as there was with twitter, github and others.  So, starting with `baseof.html` and digging through the `.html` files that it calls throughout the rest of the `layouts` dir, I discovered the `header.html` file that was processing the `menus.icons` setting that I provided in `config.toml`.  And it was running those values through another `iconlink.html` file.  

This file says that if the "Identifier" associated with this menu icon is in the list of "supported_icons" then somehow magically display the pretty icon.  

But where is the list of "supported_icons"?
```bash
$ grep -ri supported_icons themes/
# nothing 
$ find . -name "*supported*"
./themes/hugo-ficurinia/data/supported_icons.yml
```

Bingo.  `supported_icons.yaml` has the full list of icons available to me, and also the hex code for how they are rendered.  This particular font family that is being used here is called "symbols-nerd-font" (I know that because I looked at the font in the `assets/scss/style.scss` file and found:  `$symbols_font: "Symbols Nerd Font";`).

So I went to [symbols nerd font webpage](https://www.nerdfonts.com/cheat-sheet) and searched for discord icon hex code.  To actually implement the discord icon in my site, I copied up the original `supported_icons.yaml` from the theme into my base directories `data/` directory, overriding the theme's version.  Then I added this line to the file:
```bash
discord: "&#xfb6e;"
```
And added the section in the "menu.icons" section of `config.toml`, and we now have a discord icon.  

And as a good citizen, [submit a pr to upstream project](https://gitlab.com/gabmus/hugo-ficurinia/-/merge_requests/6) to include this icon by default.

# Conclusion

In conclusion, hugo is cool.  But moving on... 

# Netlify FYI

Is one way really nice hosting option for your website.  You create an account, connect it to your github account/repository, and it builds and publishes your website straight from source.  You can register a domain here too.  Publish test sites from branches of your repo, just a merge to your main branch publishes the new content.  
