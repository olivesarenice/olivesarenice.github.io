---
title: Making my digital brain
tag: 
category: others
---

## The problem

I consume alot of information everyday - some useful, some not, some I would like to keep for future reference. But I forget things easily as well. Out of all the thousands of books, articles, videos, tutorials, and blog posts I've seen, perhaps only 20% truly make an impact on me, and only 2% (that's less than 50 items) are stuck actively in my mind. The other 98% get dumped into my sub-conscious hard drive waiting for something to trigger it only for me to go: "I remember reading about that somewhere..."

For a long time, I used to subscribe to the quote:

> "I cannot remember the books I've read any more than the meals I have eaten; even so, they have made me.” — Ralph Waldo Emerson

And while that did allow me to explore freely and quickly, it wasn't effect for learning because there was no way to extract, process, and store the essential points from each piece of content. 

More importantly, there was no central platform where all kinds of content could be processed, regardless of topic, medium, or size.

## Past attempts at rememebering

I did try (and fail) to document important points everytime I consumed content, but it was too ineffective, because I frankly couldn't keep track of where everything was.

To give an example of what I've done so far:

- My project related notes would be in a local PC folder as notebooks or powerpoints.
- Portfolio works would be in PDFs uploaded onto Google Drive.
- Random writing would be in a personal OneNote section
- Book reviews for each book I read would be in a single page of OneNote
- Saved snippets from podcasts are on a specially created Twitter account
- Interesting links and articles are either in my browser bookmarks or Telegram 'Saved Messages'.
- Youtube and blog tutorials are just unrecorded because I view them on the move and have no good way to document right then.
- A Medium page for personal portfolio projects

So you see how this gets out of hand really quickly.

## What's different?

Using a website as a personal blog is nothing groundbreaking for me, but the workflow I've decided to adopt for the site is.

I realised that my content consumption patterns are so varied, sporadic, downright frenetic sometimes, that I needed a pipeline that allows me to:
1. Log ideas from what I've consumed from anywhere, anytime
2. Review and edit my log quickly and conveniently
3. Prioritize which ideas to re-visit and expand on in the future
4. Publish various forms of writeups with flexibility in code and media embedding
5. Customise the site to a certain extent without having to meddle with the full build process
6. Ideally, not have to host my own webservers to run the website.

## Components

With those requirements in mind, I decided to combine the following services:
1. Jekyll for theme-based website customisation and building
2. Github Pages for web deployment (via Github Actions) and hosting
3. Git (run locally) for managing website file versions
4. Airtable for idea logging, embedded into website.

{% include img-wrap name="1.drawio.svg" caption="Website components" %}

With these components, my workflow would be much more efficient and convenient. Most importantly, drafts (ideas) and actual projects are located in the same location (site) while still having a clear separation. 

When drafting posts, I also get to dynamically reload and preview all changes to the website locally before deployment due to the Jekyll ```serve``` function. 

## Moving forward

In the past 2 days since I've started this workflow, I've found that the Airtable web form makes it so much easier to log my ideas on the go. I no longer have to decide what is the most appropriate platform for the content I'm viewing.

The use of Github Pages is forcing me to learn how to use Git properly, and having to do actual HTML/ Markdown editing in VSCode is great practice for future software development. 

Overall, this fits all my needs with lots of room for future expansion. 

*That's all for tonight, ciao.*


