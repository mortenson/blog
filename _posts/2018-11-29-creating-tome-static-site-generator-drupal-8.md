---
layout: post
title:  "Creating Tome, a static site generator for Drupal 8"
date:   2018-11-29 08:00:00 -0500
category: drupal
---
Six months ago I started work on [Tome], a static site generator for Drupal 8.
After lots of rewrites and long nights, Tome has finally reached the beta phase
of testing and development! 🎊

Up until now, I haven’t invested a lot of time in communicating what I’m doing,
why I made Tome, or why static Drupal is hard, so now seems like a good time to
stop and reflect on things before I write more code.

## In the beginning, there was HTML.

For the unaware, static sites serve files to clients with no dynamic content.
That means that when a user visits a static homepage, they’re just being served
an “index.html” file. If you’ve ever had to write HTML files from scratch, this
might sound like a step backwards. But don’t fret! This is where static site
generators come in - they’re tools that use a structured data source to
generate a site, no Dreamweaver required. The end result is still HTML, but the
way we build static sites now is much cleaner than it used to be.

The tech behind static site generators has come a long way, but it’s still odd
to see them gain traction in an era where it’s easier than ever to create a
dynamic web page. I think this is a result of fatigue over hosting complexity,
site updates, and performance problems. Consider what it takes to host a large
scale dynamic site today - the stack isn’t trivial for individuals to set up
anymore. Even if hosting is figured out, you still have to rush to get security
updates on production, and need a site that can quickly respond to uncached
requests. For sites that could function with no or very little dynamic
behavior, static hosting is becoming an attractive alternative.

## Finding my niche.

Before building Tome I was using static site generators like Jekyll and
GatsbyJS, but still really liked Drupal 8 - the APIs, theme layer, structured
content - it felt like I would lose so much by moving out of the ecosystem.

So I started thinking - for all of the ambitious, dynamic, authenticated Drupal
sites out there, how many Drupal sites could function statically? As I looked
around, I started to feel more and more confident that static Drupal is a
viable alternative to the traditional stack. Even for sites that looked
complex, most of the interactive parts of the page were anonymous forms and
search, which can both be done statically. I had convinced myself, this is a
problem worth tackling!

I think that most sane people at this point would start by making a faster
version of `wget --recursive`, but I wasn’t satisfied with just handling static
HTML. I wanted to generate *static content* too.

## What’s static content?

Most other static site generators build from a file-based source like a
Markdown file, and I needed to do something similar for Tome. If you could
build your Drupal site from scratch, you would never need a persistent Drupal
stack running, not even on your local machine. Just edit content locally,
commit your changes to git, turn off the Drupals, and spin it all back up when
you need to.

After awhile, I realized that the closest thing we have to this in Drupal 8 is
the configuration API. Many sites already stage configuration changes in
version control in a similar way to how features worked in Drupal 7. This way
they can “check in” changes they make locally in development and have a
consistent way to get changes on production. What if you could do this for
content?

With configuration in mind, I started making a content exporter and importer
system that automatically syncs all content changes to local JSON files, and
allows users to rebuild their entire Drupal site from scratch. If you edit an
Article locally when using Drupal, that change already shows up in Git and is
ready for commit.

This gives users a completely new way to manage their content. With Git, many
workflows that are hard to accomplish normally become trivial. Need content
staging? Make a branch. Need to revert to an older version of your site?
Checkout an old commit and rebuild. Have multiple editors? Give each a fork and
only allow editors to accepts merge requests.

Beyond the content workflows introduced here, it also means that you don’t need
to run Drupal persistently, which is also new. The reduced risk of getting
hacked, the speed of doing everything locally, the cost effectiveness, there’s
a lot to be excited about.

This is sounding kind of pitch-y, which I didn’t intend it to be, so I’ll admit
that this was not easy to accomplish, and introduced a boatload of technical
complexity to Tome. Here’s a shortlist of problems tackled when building this:

1. Making exports and imports performant - Tome uses concurrency (not serial
Drupal-style batching), so I had to build a content dependency tree, sort it,
and figure out what can be imported concurrently without causing errors. For
example, you can usually import all files concurrently, but you can’t import a
node before importing the file it references.
1. Removing entity IDs from exports - if two content editors are working on two
branches, they’re bound to create entities with the same ID. Tome converts all
entity IDs to UUIDs, which prevents these collisions, but required a lot of
custom Normalizers.
1. Supporting multilingual - this is always hard to accomplish, and for Tome it
was no different. But it works now!

At the end of the day, even with all the insane code it took to get static
content working, I’m glad I pushed through and got it running. Users are
already coming up with ways to use this feature that I never thought of, like
using the exported JSON in GatsbyJS to build HTML super-duper-fast, and I’m
excited to see people continue to try it out (as long as you share what you’re
doing with me!).

## Realizing that multiple audiences exist.

After I got the static content portion of Tome working, I started to talk to
other developers and site maintainers about what I was doing, and quickly
realized that there are really three distinct audiences for Tome:

1. I want to build my site from scratch config, content, and files, then generate
static HTML locally or on a CI like Netlify (this was my original vision for
Tome).
1. I like having my content editors log into a central site, and just want Tome to
generate static HTML.
1. I want to build my site from scratch or archive it with Tome, and don’t care
about static HTML.

I was really not ready for this feedback at the time. In my head everyone would
have the same needs as me, but in reality people are unpredictable, and Drupal
site maintainers like the way things work now, they just don’t like that
production is slow, insecure, and expensive. This feedback was tough to take,
but I realized that I needed to find a solution that works for everyone.

In the context of these audiences, I decided that Tome should be split into two
sub-modules:

- Tome Sync - Automatically sync content, config, and file changes to your
filesystem, and rebuild Drupal from scratch.
- Tome Static - Generate Static HTML for your Drupal site.

With this re-architecture, users could choose the parts of Tome they want,
without being locked into the entire suite. Good for the user, a little tough
for me (as it often is).

## Static HTML, it just works right?

Tome Static was also complicated to develop, but is a lot less interesting to
talk about than Tome Sync. When users generate static HTML with Tome, they just
expect it to work, there isn’t a gradient of acceptability there.

The hardest part of Tome Static for me was making it fast. Really, really fast.
Like Tome Sync, there were many technical problems to solve, namely:

1. Getting a list of all paths on your site - Tome gets this list by loading all
content entities and routes, which I think is obvious to most people, but what
if you have thousands of content entities? I can’t rightfully load them all
without running out of memory, so I had to design a path placeholder system
that allows entity paths to be lazy-loaded at request time. I also had to
support multilingual - that means loading every translation of every content
entity, and prefixing every route with every language URL prefix. This is a
great example of a problem that seems simple at first, but turns into a huge
headache at scale.
1. Processing multiple requests in one Drupal bootstrap - This is something Drupal
doesn’t normally do, but should be possible now that we’re using Symfony’s
HttpKernel component. In practice there are a number of technical problems Tome
has to overcome, usually related to core and contrib assuming that they can
statically cache information that’s request dependent. I wouldn’t be surprised
if bug reports related to this come in after the beta releases, but we need to
process as many paths per bootstrap as possible to speed up build times.
1. Caching - Like the Page Cache module, Tome Static implements its own cache
backend that helps determine what paths need to be re-generated. It took me a
long time to find a good way to implement this, and in the end I just went with
core’s cache APIs. It would have been extremely hard to accomplish Tome’s
caching if I wasn’t using Drupal 8 (great job, everyone!).

As I mentioned before, users don’t really care how hard this part of Tome was
to build, they just expect it to work. That’s a good thing! It’s easier to
solve problems for me if it’s obvious when what I’m doing doesn’t work.

## I’m not done yet!

Even though I’ve done on a lot to get here, I already know there’s so much more
for Tome to accomplish. While the first few betas will be about making what
Tome does today better, I think the biggest tasks going forward are related to
training, documentation, and marketing. A shortlist for me is:

- Document the beta release (I’m behind on this!).
- Documenting how search and forms can work with a static Drupal site.
- Figure out what a secure edit domain looks like when only using Tome Static -
this may involve working with Drupal hosting providers and documenting a DIY
approach.
- Making Tome’s integration with static hosting platforms like Netlify richer -
for example, I want to ship an example project that uses AWS Lambda to provide
dynamic experiences while running Drupal statically.
- Think of what using Tome Sync by itself looks like - what are the use cases,
who is the audience? This isn’t completely clear to me right now.
- Revamping [https://tome.fyi](https://tome.fyi) to look better (I’m not a
designer, sorry!) and show off more of what Tome can do.

For now, for anyone who made it to the bottom of the post, I would suggest
reading through [the getting started guide] and trying out Tome for yourself.

I’m relying on users to come up with the really cool stuff for Tome, and want
to develop in the direction you’re all heading with static Drupal. Make me proud!

[Tome]: https://tome.fyi
[the getting started guide]: https://tome.fyi/docs/getting-started
