---

title: "Blanche: A Spotlight Search for Chrome"
date: 2021-01-13T15:20:41-05:00
weight: 1
aliases: ['/project']
tags: ['projects']
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: false
TocOpen: false
draft: false
hidemeta: false
disableShare: false
comments: false
---

One thing that's really been on my mind for the last semester is just how differently people are starting to view their computers/devices now in comparison to just 10+ years ago. It seems like for the early half of the 21st century, we've always forced software and hardware to adapt to the user. Today, I think the tide is starting to change; I think that people are now increasingly expected to adapt to the technology available, not the other way around. As we'll see, I think this change in expectations could be used to make pretty large changes in the way we interact with the web.  

### A quick backstory

When I was a kid, I remember watching Steve Jobs unveil the first iPhone, releasing the first smartphone with almost no buttons. The beauty was in the fact that you no longer had to learn or remember anything about how to use your phone. You didn't have to remember where special buttons were, how to use a joystick, etc., since all of that responsibility was sent over to application developers. Anyone who didn't know how to use a phone could just look at the pictures, follow the instructions, and be totally fine. Even to unlock the phone, they made it completely clear about how to do that with the classic "Slide to unlock".

If we compare the lock screens with the iPhone today, we can how vague and unclear it is to understand how to unlock your iPhone in comparison to 10 years ago. Additionally, there are way more hidden gestures such as control panel, notification center, widget panel, camera, etc. that users are **expected to remember.** 

![iphone-comparison](/img/iphone-comparison.png)

Somewhere along the way, Apple had started to expect that people would "just learn". Is this a bad thing? Definitely not. It means that tech is adapting to a new customer: one that knows what technology looks like and has expectations already. But for some reason, a lot of this expectation has only really been shifted in the mobile phone industry. After decades of computers, the average person in the United States now knows how to use a computer and how to type. Yet, unlike the mobile phone industry, the web hasn't changed to account for that. A lot of websites still depend on people to look at the screen for instructions and click on things with their trackpad which actually really slows a person down. 

### Web hasn't really changed

Chrome and the web still lives in a world where the customer is someone who has no expectations about the site they are entering. And a result, people spend way too much time clicking instead of typing, which they now know how to do much faster. 

Are there solutions? Kind of. On the far end, there is Vimium which encourages chrome users to memorize a bunch of shortcuts so they completely ditch the trackpad for the keyboard. The only catch: you have to memorize a bunch of shortcuts in advance. Not to mention, vimium gets super complicated on sites with search bars that autofocus or things with forms since you end up having to *check* to see if you can use a shortcut before you use it. What if you only had to memorize one command, and this command is guaranteed to work everywhere?

Spotlight search on Mac does this really well. Instead of telling you to memorize how to switch apps, open new apps, click through finder, etc., all you have to do is type what you're thinking and pray it becomes a result. And so, I think this is the solution we're looking for. I wanted to start really simple, so I decided to make [Blanche](https://github.com/karthikmacherla/blanche), a chrome extension that allows you to switch tabs, open recent history, and open bookmarks without ever leaving your keyboard by simulating spotlight search. Below's an image of the finished product! I'll leave a short outline about how everything works at the bottom and lessons I learned from making my first chrome extension 😀.

![demo](/img/header.png)

But ultimately, I wanted to see if there was a reason to think bigger. After using Blanche for awhile, I actually found that it made my life so much better. Imagine if every domain by had a default command/web standard that let you spotlight search and find the page/action you're looking for. No more clicking through dropdowns, exploring links, any of that. All you'd have to do is just type, search, and pray. This goes beyond the search bar we all love since actions themselves could also be done without ever leaving your keyboard (think opening a link on a page).

Sites like Stripe and Notion are already starting to do this, but the real solution could lie in creating a package which indexes html + defined actions within a domain and allows easy finding of content. Creating a generic search bar that "just works" everywhere could be the dealbreaker in getting people to easily understand your site vs. giving up out of laziness or confusion. If you got this far, let me know what you think! 

## Technical Side: Making Blanche
Git repo [here](https://github.com/karthikmacherla/blanche)!

Chrome extensions are pretty interesting in how limiting they are. They only allow developers to either (1) inject frontend html, css, javascript into a webpage (called content scripts) or (2) run background tasks (called background scripts) that get information from the **entire** chrome application. 

For (1), we can only inject these scripts on open websites (i.e. New Tab or chrome://* pages are off limits) and for (2), these background tasks can only interact with user granted features, like chrome history, bookmarks, etc. Since content scripts (1) can't access the big picture information that  (2) background scripts can, (1) and (2) also have a way of talking to each other through a messaging API.

At a high level, here's what I did: I wrote background scripts that could ask chrome about what's tabs were open and what recent history is available. I also wrote content scripts that were capable of adding my spotlight search bar to whatever page they were given and could ask the background script for their information before rendering it to the screen. 

### Content Scripts

To make the searchbar, I quickly realized that we can't really create a bar that hovers over the entire chrome level application because (2) doesn't really allow changing anything about the UI. Instead, I had to make it *look* like there was something hovering over chrome by injecting html onto the page that's currently active and styling it so that it looks the same regardless of what site we visit. Here's an example:

Doing this was actually pretty painful. I won't go into too much detail, but there were too important things to consider when you inject html and css to a page:

- How the page's default CSS styles **your** elements.
- How **your** CSS can affect the style of everything else.

To prevent the outside CSS from styling my elements (with cascading properties), I used a keyword called `!important` for **everything** that I was styling. This would prevent the page from forcing their defaults onto my elements. It seems like there's no better option given the fact that any page can do whatever they want including making things like their font's take extra precedence. 

To prevent the second event from happening, I used a relatively new feature in HTML5 called a `shadowDOM` which essentially does all of this encapsulation for free. All stylesheets I inject here will be encapsulated for free. 

Finally, I added a bunch of events like the user pressing the down arrow on focus, and pressing enter on a result, etc. 

### Background Scripts

This part was a bit less tedious. All I had to do was use the chrome api to get information about open tabs and windows and send this information to the content scripts through chrome's messaging API. All of this is pretty well documented in the code base!

### Putting it Together

If we take a look at the manifest below, we can kind of get an idea of how all of this fits together:

- The content scripts run on every page I visit, and when we hit `cmd+K` we trigger adding the search bar.
- The background scripts wait for events from the content scripts and respond back with the answers.

```
{
  "manifest_version": 2,
  "name": "Blanche",
  "description": "A spotlight search extension for chrome",
  "version": "1.0",
  "icons": {
    "128": "assets/blanche-logo.png"
  },
  "permissions": [
    "activeTab",
    "tabs",
    "history"
  ],
  "content_scripts": [
    {
      "css": [
        "style/base.css"
      ],
      "js": [
        "content.js"
      ],
      "matches": [
        "<all_urls>"
      ]
    }
  ],
  "background": {
    "scripts": [
      "background.js"
    ],
    "persistent": false
  },
  "browser_action": {
    "name": "Toggle spotlight search"
  },
  "web_accessible_resources": [
    "templates/*",
    "style/*",
    "searchBar.js",
    "assets/*"
  ],
  "commands": {
    "open-search-bar": {
      "suggested_key": {
        "default": "Ctrl+K",
        "mac": "Command+K"
      },
      "description": "Toggle spotlight search"
    },
    "_execute_browser_action": {
      "suggested_key": {
        "windows": "Ctrl+K",
        "mac": "Command+K"
      }
    }
  },
  "content_security_policy": "script-src 'self' 'sha256-/13BBW2yQVtpCsBV7JiO23y7pwEFFUobOzefJ27Nltg='; object-src 'self'"
}
```

# Closing Thoughts

Can spotlight search be the future of the web? Should we be trying to shift the tide of simplicity in favor of "ease of use"/efficiency? 